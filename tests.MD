---
created: 2026-07-09
updated: 2026-07-09
status: draft
---

# Recette — MàJ Artifactory sans impact sur Nexus IQ Firewall

## Objectif

Prouver que la mise à jour d'Artifactory n'a **pas** dégradé la chaîne de protection Nexus IQ Firewall ni la résolution de dépendances. Le test critique : un composant en violation de policy doit **toujours** être mis en quarantaine et **ne jamais** être servi par Artifactory.

## Contexte : serveur peu utilisé

Ce serveur est faiblement sollicité (**2 composants bloqués en 1 an**). Conséquence directe sur la stratégie de recette :

- Les tests de base (résolution nominale) ont déjà été faits et passent. **Mais ils ne prouvent pas que le blocage fonctionne** : le chemin de quarantaine est justement celui qu'un serveur peu utilisé n'exerce presque jamais, donc le plus susceptible de casser en silence.
- « Ça résout, donc ça marche » est **faux** ici. Il faut **au moins un blocage observé après la MàJ** pour valider.
- Deux leviers de test actif remplacent le trafic organique : (a) **rejouer les 2 composants réellement bloqués** cette année, (b) **provoquer une quarantaine neuve** avec un composant canary jamais résolu. Voir Tests 5 et 6 ci-dessous.

## Architecture testée

```
CI (GitLab)  →  Artifactory (dépôt remote/proxy)  →  Nexus IQ Firewall  →  dépôt public
                        ↑ cache local                     ↑ évaluation + quarantaine
```

Artifactory expose des dépôts remote dont l'URL amont pointe vers l'endpoint proxy du Firewall (au lieu du dépôt public direct). Le Firewall évalue chaque composant demandé : conforme → servi et mis en cache par Artifactory ; en violation → quarantaine, Artifactory renvoie une erreur.

Les 4 points de rupture possibles après une MàJ Artifactory :

1. **Config du remote repo** — URL amont vers le Firewall ou credentials réinitialisés.
2. **TLS / mTLS** — certificat client ou trust store vers le Firewall cassé.
3. **Passage en quarantaine** — un composant violé serait servi/caché malgré le Firewall (risque majeur).
4. **Cache** — composant déjà quarantiné « débloqué » par un reset de cache.

---

## Pré-requis (à figer AVANT / juste après la MàJ)

- [ ] Noter la version d'Artifactory (avant → après) et la version de Nexus IQ Firewall.
- [ ] Vérifier l'accès aux 3 dépôts remote proxifiés : `maven-remote`, `npm-remote`, `pypi-remote` (adapter aux noms réels).
- [ ] Disposer d'un compte technique / token Artifactory pour les tests.
- [ ] Avoir un accès lecture à la console Nexus IQ (dashboard Firewall / reports).
- [ ] **Retrouver les 2 composants réellement bloqués cette année** (console IQ → Firewall → historique quarantaine). Noter leur coordonnées exactes (écosystème, nom, version). Ce sont tes cas de test prioritaires, alignés sur ta vraie policy.
- [ ] Choisir **1 composant sain** par écosystème utilisé (conforme, pour valider le chemin nominal).
- [ ] Choisir **1 composant canary** en violation, **jamais résolu** chez toi, que la policy bloquera (pour un test de quarantaine « neuf » de bout en bout).

> Composant réellement bloqué = meilleur cas de test (réel, aligné policy). Canary = filet de sécurité si les 2 composants connus sont déjà en cache (leur re-demande ne repasserait pas par le Firewall).
>
> Exemples de canary (à valider contre TA policy) :
> - Maven : `org.apache.logging.log4j:log4j-core:2.14.1` (Log4Shell, CVE-2021-44228)
> - npm : `event-stream@3.3.6` (incident malware historique)
> - PyPI : `pyyaml==5.3.1` (CVE-2020-14343)

Variables utilisées dans les commandes ci-dessous :

```bash
ARTIFACTORY=https://artifactory.mon-entreprise.com/artifactory
IQ_SERVER=https://iq.mon-entreprise.com
USER=svc_recette
TOKEN=xxxxxxxx        # token Artifactory
```

---

## Checklist de recette

| # | Test | Attendu | Obtenu | OK ? |
|---|------|---------|--------|------|
| 1 | Config & connectivité Artifactory ↔ Firewall | 200 / composant caché servi | | ☐ |
| 2 | Composant sain non caché (1 par écosystème utilisé) | Servi + caché, requête visible dans Firewall | | ☐ |
| 3 | **Rejeu des 2 composants réellement bloqués** | **Toujours bloqués + quarantaine visible dans IQ** | | ☐ |
| 4 | **Quarantaine « neuve » avec canary jamais résolu** | **Intercepté à la volée, bloqué, quarantaine dans IQ** | | ☐ |
| 5 | End-to-end GitLab CI (build + éval Nexus IQ) | Report identique au baseline | | ☐ |

> Tests 3 et 4 = cœur de la recette sur ce serveur peu utilisé. Le Test 4 est **le minimum vital** : c'est le seul qui exerce le chemin complet Artifactory → Firewall → évaluation → blocage.

---

## Détail technique des tests

### Astuce commune : forcer le passage par le Firewall

Le Firewall n'est sollicité que pour un composant **non présent dans le cache** Artifactory. Deux options :

- utiliser une version jamais résolue chez toi, **ou**
- purger le composant du cache Artifactory via l'API REST :

```bash
# Purge d'un artefact caché pour forcer un re-download via le Firewall
curl -u "$USER:$TOKEN" -X DELETE \
  "$ARTIFACTORY/maven-remote-cache/org/apache/logging/log4j/log4j-core/2.14.1/"
```

(le suffixe `-cache` sur un dépôt remote contient les artefacts déjà téléchargés.)

---

### Test 1 — Config & connectivité

```bash
# Ping système Artifactory
curl -s -u "$USER:$TOKEN" "$ARTIFACTORY/api/system/ping"        # -> OK

# Lire la config du remote repo et vérifier l'URL amont = endpoint Firewall
curl -s -u "$USER:$TOKEN" "$ARTIFACTORY/api/repositories/maven-remote" | jq '.url'
# Attendu : l'URL pointe vers le proxy Nexus IQ Firewall, pas vers repo1.maven.org

# Servir un composant DEJA caché (ne doit PAS solliciter le Firewall)
curl -s -o /dev/null -w "%{http_code}\n" -u "$USER:$TOKEN" \
  "$ARTIFACTORY/maven-remote/<groupe/artefact/version/artefact-version.jar>"   # -> 200
```

Vérifie aussi le TLS si mTLS : `curl -v` doit négocier le handshake vers l'amont sans erreur de certificat côté Artifactory (logs `artifactory-service.log`).

---

### Test 2 — Composant sain non caché (chemin nominal)

L'objectif : le composant transite par le Firewall, est évalué conforme, servi puis caché. **Vérifie ensuite dans Nexus IQ que la requête est bien tracée** (dashboard Firewall → « Component requests »). Un seul écosystème suffit si le temps manque, mais idéalement un composant sain par techno réellement utilisée.

**Maven / Gradle**

```bash
# settings.xml pointant Artifactory en mirror (extrait)
# <mirror><id>art</id><mirrorOf>*</mirrorOf>
#   <url>$ARTIFACTORY/maven-remote</url></mirror>

# Récupère UN artefact sain non caché (sans polluer un vrai projet)
mvn dependency:get \
  -Dartifact=com.google.guava:guava:33.2.1-jre \
  -DremoteRepositories=art::::$ARTIFACTORY/maven-remote \
  -s settings.xml
# Attendu : téléchargement OK
```

Gradle : configurer `repositories { maven { url "$ARTIFACTORY/maven-remote" } }` puis `./gradlew dependencies`.

**npm**

```bash
# .npmrc de test
echo "registry=$ARTIFACTORY/api/npm/npm-remote/" > .npmrc
echo "//artifactory.mon-entreprise.com/artifactory/api/npm/npm-remote/:_authToken=$TOKEN" >> .npmrc

# Télécharge sans installer dans un projet
npm pack lodash@4.17.21 --userconfig ./.npmrc      # Attendu : .tgz récupéré
```

**PyPI**

```bash
pip download requests==2.32.3 --no-deps \
  --index-url "https://$USER:$TOKEN@artifactory.mon-entreprise.com/artifactory/api/pypi/pypi-remote/simple/" \
  -d ./tmp-pip
# Attendu : wheel téléchargé
```

---

### Test 3 — Rejeu des 2 composants réellement bloqués (non-régression réelle)

Reprends les 2 composants quarantinés cette année et redemande-les via Artifactory, chacun avec la commande de son écosystème. Attendu : **toujours bloqués**. Ce test prouve d'un coup que la quarantaine tient **et** que le cache n'a pas « débloqué » un composant lors de la MàJ.

```bash
# Exemple Maven (adapter au type réel de chaque composant bloqué)
mvn dependency:get -Dartifact=<groupe:artefact:version-bloquée> \
  -DremoteRepositories=art::::$ARTIFACTORY/maven-remote -s settings.xml
# Attendu : ECHEC de résolution

# Exemple npm
npm pack <paquet-bloqué>@<version> --userconfig ./.npmrc      # Attendu : échec

# Exemple PyPI
pip download <paquet-bloqué>==<version> --no-deps \
  --index-url "https://$USER:$TOKEN@artifactory.mon-entreprise.com/artifactory/api/pypi/pypi-remote/simple/" \
  -d ./tmp-pip                                                 # Attendu : échec
```

> Limite : si ces composants sont déjà en cache Artifactory, leur re-demande peut être servie/refusée **sans** re-solliciter le Firewall. Ce test valide donc surtout la persistance de la quarantaine côté Artifactory. Pour exercer le Firewall lui-même, enchaîne avec le Test 4.

### Test 4 — Quarantaine « neuve » avec un canary (LE test critique)

Le seul test qui exerce le chemin complet **Artifactory → Firewall → évaluation → blocage**. Prends un composant en violation **jamais résolu** chez toi (sinon purge-le du cache, cf. astuce plus haut).

```bash
# Maven
mvn dependency:get -Dartifact=org.apache.logging.log4j:log4j-core:2.14.1 \
  -DremoteRepositories=art::::$ARTIFACTORY/maven-remote -s settings.xml
# Attendu : ECHEC + composant intercepté et quarantiné par le Firewall

# ou npm
npm pack event-stream@3.3.6 --userconfig ./.npmrc            # Attendu : échec

# ou PyPI
pip download pyyaml==5.3.1 --no-deps \
  --index-url "https://$USER:$TOKEN@artifactory.mon-entreprise.com/artifactory/api/pypi/pypi-remote/simple/" \
  -d ./tmp-pip                                                # Attendu : échec
```

**Vérification côté Nexus IQ** — la quarantaine doit être visible pour les Tests 3 et 4 :

- Console IQ → **Firewall** → composant listé comme *Quarantined*.
- Ou via l'API REST Firewall :

```bash
curl -s -u "$USER:$TOKEN" \
  "$IQ_SERVER/api/v2/firewall/quarantine?repositoryId=<id-repo-firewall>" | jq .
```

> ⚠️ Si un composant en violation est **servi malgré tout** (200 + fichier récupéré), c'est un **échec bloquant** : la MàJ a cassé le chemin de quarantaine. Ne pas mettre en production.

En parallèle, vérifie qu'un composant sain déjà caché reste servi normalement (Test 1) sans latence anormale.

---

### Test 5 — End-to-end GitLab CI

Rejouer un vrai job qui résout ses dépendances depuis Artifactory puis évalue avec Nexus IQ, et **comparer au baseline pris avant la MàJ** (mêmes composants, mêmes violations, même verdict de policy).

`.gitlab-ci.yml` (extrait) :

```yaml
stages:
  - resolve
  - iq-eval

variables:
  ARTIFACTORY: "https://artifactory.mon-entreprise.com/artifactory"

resolve:
  stage: resolve
  script:
    - mvn -s ci_settings.xml clean package    # dépendances via Artifactory -> Firewall
  artifacts:
    paths:
      - target/

nexus-iq:
  stage: iq-eval
  script:
    - >
      java -jar nexus-iq-cli.jar
      -i "$IQ_APP_ID"                # application ID dans Nexus IQ
      -s "$IQ_SERVER"
      -a "$IQ_USER:$IQ_TOKEN"
      -t build
      target/
  # Attendu : rapport de policy identique au baseline (nb composants, violations)
```

**Comparaison baseline** : récupérer le report d'un build de référence via l'API IQ et diff avec le nouveau.

```bash
# Résumé d'évaluation d'une application (rapports successifs)
curl -s -u "$IQ_USER:$IQ_TOKEN" \
  "$IQ_SERVER/api/v2/reports/applications/$IQ_APP_ID" | jq '.[].reportHtmlUrl'
```

Un écart de composants résolus ou de violations détectées = régression à investiguer.

---

## Critères Go / No-Go

**GO** si, et seulement si :

- Tests 1–2 OK (résolution nominale fonctionnelle),
- Test 3 OK = les 2 composants réellement bloqués le restent (non-régression du cache),
- **Test 4 OK = le canary est intercepté et quarantiné à la volée** (chemin Firewall complet validé) — condition impérative,
- Test 5 : report CI identique au baseline.

**NO-GO** (bloquant) si le canary (Test 4) est servi par Artifactory, si un composant en violation n'apparaît plus en quarantaine dans Nexus IQ, ou si le Test 4 n'a pas pu être exécuté (aucun blocage observé après la MàJ = recette non concluante sur ce serveur peu utilisé).

---

## Notes

- Adapter tous les noms de dépôts (`maven-remote`, `npm-remote`, `pypi-remote`) et l'`IQ_APP_ID` à ton environnement.
- Idéalement, exécuter la recette sur un **environnement de pré-prod** miroir avant de valider la prod.
- Conserver les sorties (logs, codes HTTP, captures IQ) dans la colonne « Obtenu » comme preuve de recette.
