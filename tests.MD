---
created: 2026-07-09
updated: 2026-07-09
status: draft
---

# Tests — MàJ Artifactory sans impact sur Nexus IQ Firewall (npm-only)

## Objectif

Prouver que la mise à jour d'Artifactory n'a pas cassé la chaîne de protection Nexus IQ Firewall sur le remote **npmjs**. Test critique : un paquet npm en violation doit **toujours** être mis en quarantaine et **ne jamais** être servi par Artifactory.

## Architecture

```
CI / poste  →  Artifactory (npm-remote, proxy)  →  Nexus IQ Firewall  →  registry.npmjs.org
                     ↑ cache local                      ↑ évaluation + quarantaine
```

Seul le remote **npm** est proxifié par le Firewall. Les tests ne portent donc que sur npm.

## Contexte : serveur peu utilisé

2 paquets bloqués en 1 an. Les tests nominaux passent déjà mais ne prouvent **pas** que le blocage fonctionne — c'est justement le chemin le moins exercé. Il faut **au moins un blocage observé après la MàJ** pour valider (Test 3 = canary).

## Variables

```bash
ARTIFACTORY=https://artifactory.mon-entreprise.com/artifactory
IQ_SERVER=https://iq.mon-entreprise.com
USER=svc_recette
TOKEN=xxxxxxxx        # token Artifactory
NPM_REMOTE=$ARTIFACTORY/api/npm/npm-remote/   # adapter le nom du repo
```

Fichier `.npmrc` de test (à créer dans un dossier vide) :

```bash
cat > .npmrc <<EOF
registry=$NPM_REMOTE
//artifactory.mon-entreprise.com/artifactory/api/npm/npm-remote/:_authToken=$TOKEN
EOF
```

> `npm pack` (et non `npm install`) : télécharge juste le tarball, n'exécute aucun script de lifecycle. Si le Firewall fait son travail, le paquet malveillant n'atteint jamais la machine.

---

## Checklist

| # | Test | Attendu | Obtenu | OK ? |
|---|------|---------|--------|------|
| 1 | Connectivité + paquet sain non caché | `.tgz` récupéré, requête visible dans Firewall | | ☐ |
| 2 | Rejeu des 2 paquets réellement bloqués | Toujours bloqués + quarantaine visible dans IQ | | ☐ |
| 3 | **Canary jamais résolu (`event-stream@3.3.6`)** | **Bloqué (403/404), quarantaine à la volée dans IQ** | | ☐ |

> Test 3 = minimum vital : seul test qui exerce le chemin complet Artifactory → Firewall → blocage.

---

## Détail des tests

### Astuce — forcer le passage par le Firewall

Le Firewall n'est sollicité que pour un paquet **absent du cache**. Purge si besoin :

```bash
curl -u "$USER:$TOKEN" -X DELETE \
  "$ARTIFACTORY/npm-remote-cache/event-stream/"
```

### Test 1 — Paquet sain non caché (chemin nominal)

```bash
npm pack lodash@4.17.21 --userconfig ./.npmrc
# Attendu : lodash-4.17.21.tgz récupéré
```

Vérifier dans la console IQ → Firewall → « Component requests » que la requête est tracée.

### Test 2 — Rejeu des 2 paquets réellement bloqués (non-régression)

Reprendre les 2 paquets quarantinés cette année (console IQ → Firewall → historique) et les redemander :

```bash
npm pack <paquet-bloqué>@<version> --userconfig ./.npmrc
# Attendu : ECHEC (toujours bloqué)
```

Prouve que la quarantaine tient et que le cache n'a pas « débloqué » un paquet lors de la MàJ.

> Limite : s'ils sont déjà en cache, la demande peut être refusée sans re-solliciter le Firewall. D'où le Test 3.

### Test 3 — Canary (LE test critique)

Paquet en violation jamais résolu → exerce le chemin complet. `event-stream@3.3.6` est flagué *malicious* par Sonatype (incident malware 2018), donc quarantaine quasi garantie quelle que soit la policy.

```bash
# S'assurer qu'il n'est pas caché (cf. purge ci-dessus), puis :
npm pack event-stream@3.3.6 --userconfig ./.npmrc
# Attendu : ECHEC + event-stream 3.3.6 intercepté et quarantiné dans Nexus IQ
```

Alternatives de canary si besoin : `flatmap-stream@0.1.1` (le paquet malveillant lui-même) ou `eslint-scope@3.7.2` (version compromise).

**Vérification côté Nexus IQ** (Tests 2 et 3) :

- Console IQ → **Firewall** → paquet listé comme *Quarantined*.
- Ou via l'API REST :

```bash
curl -s -u "$USER:$TOKEN" \
  "$IQ_SERVER/api/v2/firewall/quarantine?repositoryId=<id-repo-firewall>" | jq .
```

> ⚠️ Si `event-stream@3.3.6` est **servi** (200 + `.tgz` récupéré), c'est un **échec bloquant** : la MàJ a cassé la quarantaine. Ne pas mettre en production.

---

## Go / No-Go

**GO** si :

- Test 1 OK (résolution nominale),
- Test 2 OK (les 2 paquets bloqués le restent),
- **Test 3 OK = `event-stream@3.3.6` intercepté et quarantiné à la volée** — condition impérative.

**NO-GO** si le canary est servi par Artifactory, si un paquet en violation n'apparaît plus en quarantaine dans IQ, ou si le Test 3 n'a pas pu produire de blocage (recette non concluante sur ce serveur peu utilisé).

---

## Notes

- Adapter le nom du repo (`npm-remote`), l'URL et le token.
- Retrouver dans IQ les coordonnées exactes des 2 paquets bloqués (Test 2).
- Idéalement exécuter sur un environnement de pré-prod miroir avant la prod.
- Conserver les sorties (codes HTTP, captures IQ) dans la colonne « Obtenu ».
