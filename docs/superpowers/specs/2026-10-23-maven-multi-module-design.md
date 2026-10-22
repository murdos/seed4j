# Seed4J — découpage en multi-module Maven

Date : 2026-10-23
Statut : brouillon, en attente de validation

## 1. Intention

Aujourd'hui seed4j est un unique jar Spring Boot qui contient tout : l'API utilisée par les modules, les ~120 modules
officiels, le moteur d'application, le serveur REST et l'IHM Vue.

Trois douleurs motivent le découpage :

1. **La CLI est trop lourde** : elle embarque tout seed4j (Tomcat, Spring Web, springdoc, bundle Vue, statistiques)
   alors qu'elle n'a besoin que d'appliquer des modules.
2. **La CLI ne sait pas charger d'extensions** : il n'existe aucun mécanisme pour lui dire « prends aussi les modules
   de tel artefact Maven ».
3. **Développer une extension n'est pas trivial** : elle dépend du jar seed4j complet, sans surface d'API définie, et
   doit reproduire le montage d'une application seed4j entière. Avec le découpage, une extension devient une
   **simple librairie** qui dépend de `seed4j-api` et `seed4j-test`.

L'objectif est d'exposer une **API claire** pour les auteurs d'extensions, et un **moteur réutilisable** qui sait,
à partir d'une simple liste de coordonnées Maven, résoudre, charger et appliquer des modules.

## 2. Objectifs, non-objectifs, anti-objectifs

### Objectifs

- Découper le dépôt en artefacts Maven distincts (voir §4) avec des frontières vérifiées par ArchUnit.
- Une extension est une simple librairie qui ne dépend que de `seed4j-api` (+ `seed4j-test` en scope test). Si elle
  étend un module officiel, elle tire en plus `seed4j-core-modules` (pour `Seed4JCoreModuleSlug`).
- `seed4j-engine` charge lui-même les sources de modules à partir de coordonnées Maven ; l'appelant (CLI, serveur) ne
  fournit que la liste des coordonnées et, éventuellement, des options de résolution.
- La CLI peut ne dépendre que de `seed4j-engine` (Spring « light » : `spring-context`, pas de Spring Boot web).
- Le serveur bénéficie du même mécanisme de chargement d'extensions.

### Non-objectifs (chantiers ultérieurs, mais à ne pas rendre plus difficiles)

- **Abandon de l'architecture hexagonale par module** : à terme un module = une factory + un descripteur de
  métadonnées, découvert via `ServiceLoader` / `@AutoService` au lieu d'un bean Spring. Ce chantier n'est **pas** inclus
  ici, mais le design de la « source de modules » (§5) doit permettre de remplacer la découverte par beans Spring par
  `ServiceLoader` sans toucher au reste du moteur.
- Implémenter la CLI elle-même (dépôt séparé) : ce chantier livre ce dont elle a besoin.
- Support JPMS (`module-info.java`).

### Anti-objectifs (échec même si « ça marche »)

- Garder les mêmes packages répartis sur plusieurs jars (split packages).
- Un moteur qui suppose que tout est sur son propre classpath (templates ou fichiers de versions introuvables pour une
  extension chargée dynamiquement).
- Une CLI qui doit réimplémenter de la logique de résolution/chargement.
- Perte de la barre qualité actuelle : 100 % de couverture, zéro smell Sonar, règles hexagonales vérifiées.

## 3. Contraintes et décisions déjà prises

| Décision                                                                                 | Source      |
| ---------------------------------------------------------------------------------------- | ----------- |
| La compatibilité avec les extensions existantes peut être cassée ; renommage propre      | utilisateur |
| Spring « light » (`spring-context`) est acceptable dans la CLI                           | utilisateur |
| Le moteur gère lui-même le chargement des extensions                                     | utilisateur |
| Résolution depuis Maven Central, `~/.m2`, et dépôts privés configurés via `settings.xml` | utilisateur |
| L'hexagonal reste pour le moteur et le serveur ; son abandon ne concerne que les modules | utilisateur |
| Artefacts nommés `seed4j-test` et `seed4j-core-modules`                                  | utilisateur |
| `Seed4JCoreModuleSlug` vit dans `seed4j-core-modules`, pas dans l'API                    | utilisateur |

## 4. Artefacts cibles

```
seed4j-parent (pom, gestion des versions + plugins communs)
├── seed4j-api           com.seed4j.api          DSL de module + modèle + descripteurs + erreurs partagées. Pur Java.
├── seed4j-engine        com.seed4j.engine       Application des modules, landscape, historique projet, chargement de sources.
├── seed4j-test          com.seed4j.test         Assertions/fixtures pour tester un module (dépend de api + engine).
├── seed4j-core-modules  com.seed4j.modules.core  Modules officiels + Seed4JCoreModuleSlug + templates + fichiers de versions.
└── seed4j-server        com.seed4j.server       REST, OpenAPI, IHM Vue, statistiques, téléchargement, wire → jar exec.
```

Graphe de dépendances (compile) :

```
api  <──  engine  <──  server
 ^          ^            │
 │          └── test     │ (runtime : core-modules, ou chargés comme source)
 └── core-modules  <─────┘
```

Une extension : `api` (provided) + `test` (test) ; + `core-modules` (provided) si elle référence un module officiel.

### 4.1 `seed4j-api`

Contenu (aujourd'hui dans `com.seed4j.module.domain` et `com.seed4j.shared`) :

- le DSL `Seed4JModule` et tout ce qu'il expose publiquement (types de `file`, `javadependency`, `javabuild`,
  `mavenplugin`, `gradleplugin`, `gradleconfiguration`, `nodejs`, `packagejson`, `docker`, `javaproperties`,
  `properties`, `replacement`, `buildproperties`, `startupcommand`, `gitignore`, `postaction`…) ;
- les descripteurs : `Seed4JModuleResource`, `Seed4JModuleApiDoc`, `Seed4JModuleOrganization`,
  `Seed4JModuleTags`, `Seed4JModulePropertiesDefinition`, `Seed4JModuleSlugFactory`, `Seed4JFeatureSlugFactory` ;
- `shared.error.domain` (`Assert`, `Seed4JException`, clés d'erreur), `shared.collection`, `shared.base` ;
- **pas** `Seed4JCoreModuleSlug` / `Seed4JCoreFeatureSlug` : ils partent dans `seed4j-core-modules`. La constante
  `Seed4JModuleOrganization.SPRINGBOOT_DEPENDENCY`, qui les référence, part avec eux.

Dépendances autorisées : JDK, commons-lang3, commons-io (déjà utilisés par `module.domain`), slf4j-api. **Pas de
Spring.**

Mesure de départ : les générateurs importent directement 75 des 264 classes de `module.domain`, aucune classe de
`module.infrastructure`. La surface réelle est plus large (types renvoyés par le builder) ; la règle ArchUnit de
l'étape 1 (§7) en donnera la liste exacte.

### 4.2 `seed4j-engine`

- Classes moteur de `module.domain` : `Seed4JModulesApplyer`, `Seed4JModuleChanges`, `Seed4JModulesToApply`,
  `landscape`, presets, ports (`Seed4JModulesRepository`, `GitRepository`, `GeneratedProjectRepository`,
  `JavaDependenciesVersionsRepository`, `ProjectJavaDependenciesRepository`, `ProjectJavaBuildToolRepository`…).
- `module.application`.
- `module.infrastructure.secondary` entier (FileSystem, Mustache, JGit, lecteurs Maven/Gradle/npm, handlers
  YAML/properties/spring.factories, docker images).
- `project.domain.history` + sa persistance : le moteur écrit aujourd'hui l'historique via
  `project.infrastructure.primary.JavaProjects` (appelé depuis `FileSystemSeed4JModulesRepository`) ; la CLI en a
  besoin. Ce couplage infra→infra doit devenir un port du moteur.
- Chargement de sources (§5).
- Une `@Configuration` Spring publique (`Seed4JEngineConfiguration`) qui câble le moteur, importable en une ligne.
- Les événements `Seed4JModuleEvents` restent un port ; l'adaptateur Spring events reste disponible pour que le
  serveur branche les statistiques.

### 4.3 `seed4j-test`

- `Seed4JModulesAssertions`, `Seed4JModulesFixture` et helpers associés (`TestFileUtils`, projets de test), aujourd'hui
  dans `src/test` sous `module.infrastructure.secondary` / `module.domain`.
- Publié en artefact normal (pas `test-jar`) pour être consommé proprement par les extensions.

### 4.4 `seed4j-core-modules`

- `com.seed4j.generator.**` → `com.seed4j.modules.core.**`.
- `src/main/resources/generator/**` (templates) et `generator/dependencies/**` (fichiers de versions pom/gradle/npm/
  Dockerfile, toujours mis à jour par Renovate).
- Exposition à la découverte : beans Spring `Seed4JModuleResource` dans ce chantier (§5.3).
- Tests unitaires des factories, utilisant `seed4j-test`.

### 4.5 `seed4j-server`

- `module.infrastructure.primary` (REST, OpenAPI), `project` (hors historique), `statistic`, `wire`,
  `Seed4JApp`, IHM Vue (`src/main/webapp`, `package.json`, Vite, Vitest, Cypress), NativeHints, Cucumber.
- Dépend de `seed4j-engine` ; prend `seed4j-core-modules` comme source par défaut et accepte des sources supplémentaires
  par configuration (`seed4j.sources=...`).
- Produit le jar `exec` actuel.

## 5. Sources de modules

### 5.1 Notion de source

Une **source de modules** est l'unité que le moteur charge. Elle fournit :

- un ensemble de `Seed4JModuleResource` ;
- zéro ou plusieurs fichiers de versions de dépendances (Maven, Gradle version catalog, npm, Docker) ;
- un accès à ses ressources (templates) **via son propre classloader**.

Le moteur ne lit plus jamais un chemin de ressource via `FileSystemProjectFiles.class.getResourceAsStream(...)`
pour du contenu de module : il passe par la source qui a produit le module.

### 5.2 Chargement

Entrée fournie par l'appelant : une liste de coordonnées `groupId:artifactId[:version]` plus des options
(fichier `settings.xml`, mode hors-ligne).

Trois étapes, chacune derrière un port du moteur :

1. **Résolution** — coordonnées → jars locaux, transitives incluses. Implémentation par défaut : MIMA (Maven Resolver
   embarqué), qui gère nativement `~/.m2/settings.xml` (miroirs, authentification, proxies), le dépôt local
   `~/.m2/repository` et Central. Le port permet aux tests et au serveur de fournir directement des chemins de jars.
2. **Chargement** — un classloader enfant par source ; `seed4j-api` (et ses dépendances) chargé par le classloader
   parent et exclu de la résolution. Les extensions déclarent `seed4j-api` en scope `provided`.
3. **Découverte** — obtenir les `Seed4JModuleResource` de la source (§5.3).

Règles :

- slug en double entre deux sources → erreur explicite (`DuplicatedSlugException` existe déjà), nommant les deux
  sources ;
- fichiers de versions : la version déclarée par la source du module a priorité, puis celles de `seed4j-core-modules`
  (aujourd'hui fait partiellement par la liste ordonnée `@Order` de `JavaDependenciesReader`) ;
- une source dont la version d'API attendue est incompatible est rejetée avec un message clair (version lue dans le
  manifest de l'extension — détail laissé à l'implémenteur).

### 5.3 Découverte dans ce chantier vs. le suivant

- **Ce chantier** : la découverte crée un contexte Spring enfant par source, dont le parent est le contexte du
  moteur, et récupère les beans `Seed4JModuleResource`. Le point d'entrée du scan est déclaré par la source (par ex.
  une entrée `META-INF/seed4j/source.properties` donnant la classe `@Configuration` racine).
- **Chantier suivant** : la découverte devient `ServiceLoader.load(<SPI>, sourceClassLoader)` et les modules perdent
  leurs couches application/infrastructure. Seule l'implémentation du port de découverte change.

## 6. Qualité et outillage

- **Couverture** : JaCoCo par module + rapport agrégé (`report-aggregate`) pour Sonar ; objectif inchangé, 100 %.
- **ArchUnit** : `HexagonalArchTest` adapté par module ; nouvelles règles :
  - `api` ne dépend que du JDK, commons-lang3, commons-io, slf4j ;
  - `modules` ne dépend que de `api` (+ Spring context, le temps de ce chantier) ;
  - personne ne dépend de `server`.
- **Sonar** : projet multi-module.
- **Frontend** : `package.json`, Vite, Vitest, Cypress, Tikui migrent dans `seed4j-server`. Prettier-java et les
  hooks de pre-commit doivent continuer à couvrir tout le Java du dépôt (config Prettier à la racine ou dans server
  avec des chemins élargis — au choix de l'implémenteur).
- **Cucumber** : reste dans `seed4j-server` (il teste l'API REST).
- **Native** (GraalVM) : `NativeHints` suit le code qu'il décrit ; le build natif concerne `seed4j-server`. Le
  chargement dynamique de sources n'a pas à fonctionner en natif.
- **CI** : `tests-ci/generate.sh`, `.github/workflows/github-actions.yml`, `release.sh`, `Dockerfile`, chemins
  Renovate (`generator/dependencies/**` déménage dans `seed4j-core-modules`), `CLAUDE.md`.

## 7. Séquencement recommandé

Chaque étape est livrable seule et garde la CI verte.

1. **Révéler les frontières dans le module actuel** : règles ArchUnit « generator → seulement l'API » et « API → pas
   de moteur/infra/Spring », en listant les violations. Corriger les fuites (`Seed4JModuleOrganization` →
   `Seed4JCoreModuleSlug`, `FileSystemSeed4JModulesRepository` → `JavaProjects`, chemins `/generator/...` en dur).
2. **Introduire la notion de source** (§5.1) dans le code actuel, avec une seule source « classpath » : templates et
   fichiers de versions lus via la source.
3. **Renommer les packages** vers les racines cibles (`com.seed4j.api`, `engine`, `test`, `modules.core`, `server`),
   toujours dans un seul module Maven.
4. **Extraire les modules Maven** : parent, puis `api`, `engine`, `test`, `core-modules`, `server`. Outillage du §6.
5. **Chargement dynamique** (§5.2) : résolution MIMA, classloaders, découverte par contexte Spring enfant ; option de
   configuration côté serveur.

## 8. Décisions laissées à l'implémenteur / encore ouvertes

- **Nom de package** de `seed4j-core-modules` : `com.seed4j.modules.core` (décision utilisateur).
- **Versioning** — _mon choix_ : tous les artefacts partagent la version du parent et sont publiés ensemble.
- Format exact du descripteur de source (`META-INF/seed4j/source.properties`) et de la vérification de version d'API.
- Nom de la propriété de configuration côté serveur.
