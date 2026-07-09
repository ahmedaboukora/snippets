---
created: 2026-07-09
updated: 2026-07-09
status: draft
---

# Recette — MàJ Artifactory sans impact sur Nexus IQ Firewall

## Objectif

Prouver que la mise à jour d'Artifactory n'a **pas** dégradé la chaîne de protection Nexus IQ Firewall ni la résolution de dépendances. Le test critique : un composant en violation de policy doit **toujours** être mis en quarantaine et **ne jamais** être servi par Artifactory.

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
- [ ] Choisir, **par écosystème**, deux composants de référence :
  - un composant **sain** (conforme à la policy),
  - un composant **en violation connue** (attendu : quarantaine).

> Exemples illustratifs de composants « en violation » fréquemment quarantinés (à valider contre TA policy) :
> - Maven : `org.apache.logging.log4j:log4j-core:2.14.1` (Log4Shell, CVE-2021-44228)
> - npm : `event-stream@3.3.6` (incident malware historique)
> - PyPI : une version de paquet avec CVE critique connue (ex. `pyyaml==5.3.1`, CVE-2020-14343)

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
| 2 | Composant sain non caché (Maven) | Servi + caché, requête visible dans Firewall | | ☐ |
| 3 | Composant sain non caché (npm) | idem | | ☐ |
| 4 | Composant sain non caché (PyPI) | idem | | ☐ |
| 5 | **Quarantaine (Maven)** | **Bloqué (403/409), quarantaine visible dans IQ** | | ☐ |
| 6 | **Quarantaine (npm)** | **Bloqué** | | ☐ |
| 7 | **Quarantaine (PyPI)** | **Bloqué** | | ☐ |
| 8 | Non-régression cache (composant déjà quarantiné reste bloqué) | Toujours bloqué | | ☐ |
| 9 | End-to-end GitLab CI (build + éval Nexus IQ) | Report identique au baseline | | ☐ |

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

### Test 2/3/4 — Composant sain non caché (chemin nominal)

L'objectif : le composant transite par le Firewall, est évalué conforme, servi puis caché. **Vérifie ensuite dans Nexus IQ que la requête est bien tracée** (dashboard Firewall → « Component requests »).

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

### Test 5/6/7 — Quarantaine (LE test critique)

Même commandes que ci-dessus, mais avec un composant **en violation** et **non caché**. Attendu : la commande **échoue** (403/409/404), et la quarantaine apparaît dans Nexus IQ.

**Maven**

```bash
# S'assurer qu'il n'est pas caché (cf. purge plus haut), puis :
mvn dependency:get \
  -Dartifact=org.apache.logging.log4j:log4j-core:2.14.1 \
  -DremoteRepositories=art::::$ARTIFACTORY/maven-remote -s settings.xml
# Attendu : ECHEC de résolution (composant quarantiné par le Firewall)
```

**npm**

```bash
npm pack event-stream@3.3.6 --userconfig ./.npmrc
# Attendu : échec (403/404)
```

**PyPI**

```bash
pip download pyyaml==5.3.1 --no-deps \
  --index-url "https://$USER:$TOKEN@artifactory.mon-entreprise.com/artifactory/api/pypi/pypi-remote/simple/" \
  -d ./tmp-pip
# Attendu : échec
```

**Vérification côté Nexus IQ** — la quarantaine doit être visible :

- Console IQ → **Firewall** → composant listé comme *Quarantined*.
- Ou via l'API REST Firewall (composants en quarantaine) :

```bash
curl -s -u "$USER:$TOKEN" \
  "$IQ_SERVER/api/v2/firewall/quarantine?repositoryId=<id-repo-firewall>" | jq .
```

> ⚠️ Si un composant en violation est **servi malgré tout** (200 + fichier récupéré), c'est un **échec bloquant** de la recette : la MàJ a cassé le chemin de quarantaine. Ne pas mettre en production.

---

### Test 8 — Non-régression du cache

```bash
# Un composant DEJA quarantiné avant la MàJ doit rester bloqué
mvn dependency:get -Dartifact=<composant-deja-quarantine> \
  -DremoteRepositories=art::::$ARTIFACTORY/maven-remote -s settings.xml
# Attendu : toujours ECHEC (pas de "déblocage" par reset de cache)
```

Vérifier en parallèle qu'un composant sain déjà caché reste servi normalement (Test 1) sans latence anormale.

---

### Test 9 — End-to-end GitLab CI

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

- Tests 1–4 OK (résolution nominale fonctionnelle),
- Tests 5–7 OK = **tous les composants en violation sont bloqués** et quarantinés dans Nexus IQ,
- Test 8 OK (pas de déblocage par le cache),
- Test 9 : report CI identique au baseline.

**NO-GO** (bloquant) si un composant en violation est servi par Artifactory, ou si la quarantaine n'apparaît plus dans Nexus IQ après la MàJ.

---

## Notes

- Adapter tous les noms de dépôts (`maven-remote`, `npm-remote`, `pypi-remote`) et l'`IQ_APP_ID` à ton environnement.
- Idéalement, exécuter la recette sur un **environnement de pré-prod** miroir avant de valider la prod.
- Conserver les sorties (logs, codes HTTP, captures IQ) dans la colonne « Obtenu » comme preuve de recette.
