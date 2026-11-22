# Préparation : Composition en go-templating avec provider-grafana

Suite de `docs/tenant-onboarding-roadmap.md` (projet 1, étapes 2 à 6). Ce document liste quoi faire et quoi lire. Il ne contient pas de code Crossplane : c'est toi qui l'écris.

## Où on en est

Dans `tenantplatform/` :

- `apis/tenants/schema.yaml` : source SimpleSchema, ne sert plus à rien une fois la XRD générée. **Ne pas l'appliquer au cluster.**
- `apis/tenants/definition.yaml` : la XRD, source de vérité. Elle contient `scope: Namespaced`, la règle CEL d'immutabilité de `tenantId` (plage 1000–1999) et les colonnes d'affichage.
- `examples/tenant-xr-example.yaml` : le tenant `tenant-1005`, avec les trois signaux à `true`.
  Il est validé par `crossplane resource validate` **à condition que `kind` soit `Tenant` (majuscule)**. Vérifie ce point en premier demain.

Rien n'est encore déployé : `platform/` et `tenants/` sont vides, et rien n'est commité côté `tenantplatform/`, `docs/` et `CLAUDE.md`.

## Ce qu'on sait du provider Grafana

Vérifié dans le repo (pas à l'exécution) :

- Le provider est **`grafana/crossplane-provider-grafana`** (et non `crossplane-contrib`), image `xpkg.upbound.io/grafana/provider-grafana`. Dernière release vue : **v2.18.1** (9 novembre 2026). Le README est en retard (il cite v2.2.0).
- Il fournit des CRDs **namespacées** dans les groupes `.m.` :
  - `oss.grafana.m.crossplane.io`, kind **`DataSource`**, `scope: Namespaced`. C'est la ressource à composer ;
  - `grafana.m.crossplane.io`, kinds **`ProviderConfig`** (namespacée) et **`ClusterProviderConfig`**.
- Champs de `spec.forProvider` de la `DataSource` namespacée : `name`, `type`, `url`, `uid`, `orgId`, `isDefault`, `accessMode`, `basicAuthEnabled`, `jsonDataEncoded`, `secureJsonDataEncodedSecretRef`.
- La `ClusterProviderConfig` porte `url`, `orgId` et `credentials` (une source et une référence à un Secret).

Non vérifié :

- le comportement à l'exécution ;
- la compatibilité de la v2.18.1 avec ta Crossplane **2.4.3** ;
- la forme exacte de `secureJsonDataEncodedSecretRef` ;
- la façon de passer les en-têtes `AccountID` / `ProjectID` : le nom de l'en-tête va probablement dans `jsonDataEncoded` et la valeur dans le Secret, ce qui impliquerait un Secret par tenant. **À confirmer dans le CRD.**

## Ordre de travail

### 0. Lire le CRD de référence

Dans le repo du provider, le fichier `package/crds/oss.grafana.m.crossplane.io_datasources.yaml`. Réponds par écrit à ces questions avant d'écrire quoi que ce soit :

1. Quels champs sont obligatoires pour une `DataSource` ?
2. Comment exprimer un en-tête HTTP personnalisé, et sa valeur ?
3. Quel `type` pour VictoriaMetrics, VictoriaLogs et VictoriaTraces ? Les valeurs sont dans `infrastructure/controllers/grafana/helmrelease.yaml` : `prometheus`, `victoriametrics-logs-datasource` et `jaeger`.
4. Que fait `uid` ? Les dashboards existants référencent des UIDs précis.
5. Que se passe-t-il si une data source du même nom existe déjà dans Grafana ?

### 1. Décider comment donner l'accès à Grafana

- **`ClusterProviderConfig`** (recommandé : un seul Grafana pour tous les tenants) ou `ProviderConfig` par namespace.
- Il faut un **service account Grafana avec un token**, rangé dans un Secret. Pour un premier test, créé à la main. Comment il arrivera plus tard par Flux (SOPS ?) reste ouvert.
- Vérifie que le service account a le droit de créer des data sources.

### 2. Installer la plomberie dans `platform/`

- Le **Provider** grafana, en fixant la version.
- Les **Functions** : **function-go-templating** (la logique « si le signal est choisi, alors créer la data source ») et **function-auto-ready** (pour un Ready fiable, indispensable au projet 2).
- La **ClusterProviderConfig** : elle dépend du CRD du provider, donc elle doit s'appliquer après lui. Pense à l'ordre dans Flux (`dependsOn`, ou un découpage supplémentaire).
- Ne liste dans `platform/kustomization.yaml` que `definition.yaml` et la Composition, jamais `schema.yaml`.

### 3. Écrire la Composition

- Génère le squelette : `crossplane composition generate apis/tenants/definition.yaml`.
- Mode pipeline : go-templating, puis auto-ready.
- Dans le template :
  - une data source par signal activé (`spec.signals.metrics`, `.logs`, `.traces`) ;
  - un nom unique par tenant et par signal ;
  - l'`AccountID` vient de `spec.tenantId`, et `ProjectID` reste à `0` ;
  - **un label avec l'ID du tenant sur chaque ressource composée** (le projet 2 en dépend).
- Le XR est namespacé : décide dans quel namespace vivent les ressources composées.

### 4. Tester hors cluster

- Rendu : `crossplane composition render` (commande renommée en CLI v2.5). Il lance les functions dans des conteneurs **Docker** : vérifie que Docker est disponible sous WSL.
- Il faut fournir au rendu la Composition, le XR d'exemple et un fichier déclarant les functions.
- Validation : `crossplane resource validate` (ex-`beta validate`) sur la sortie du rendu, avec les schémas du provider.
- Attendu : trois data sources pour `tenant-1005`, une seule si seules les métriques sont activées, et zéro pour un signal à `false`.

### 5. Déployer et jouer les scénarios

- Dans `tenants/` : le namespace du tenant, puis le XR. Décide qui crée le namespace.
- Scénarios : création, retrait d'un signal (la data source doit disparaître), tentative de changer `tenantId` (ta règle CEL doit refuser), suppression du tenant.
- Si la XRD reste en `ESTABLISHED False`, regarde `kubectl describe xrd tenants.myobsstack.training.com` : c'est là que les erreurs CEL ou de colonnes apparaissent.

## Pièges à garder en tête

- `kubectl apply --dry-run=server` d'une XRD ne compile ni les règles CEL ni les `jsonPath` : seul le CRD généré les valide.
- Régénérer avec `crossplane xrd generate` écraserait les ajouts manuels de `definition.yaml`.
- Les paramètres de `AccountID` / `ProjectID` ne donnent pas d'isolation réelle (pas de vmauth), c'est voulu pour le lab.

## À lire

- Dépôt du provider : `github.com/grafana/crossplane-provider-grafana` (dossier `package/crds/`, et `examples/` s'il en existe pour les data sources).
- docs.crossplane.io, version 2.x : **Compositions**, **Composition Functions**, puis la page de **function-go-templating** (sur son repo GitHub et sur le marketplace Upbound).
- CLI : `crossplane composition generate --help` et `crossplane composition render --help`.
