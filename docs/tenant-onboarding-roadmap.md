# Tenant onboarding — feuille de route

Deux projets distincts qui se complètent, à faire dans l'ordre :

1. **Le tenant Crossplane** : l'orchestration de l'onboarding.
2. **L'observabilité de l'onboarding** : voir dans Grafana comment se passe chaque onboarding.

Le but du lab est d'apprendre Crossplane v2 en orchestrant un tenant.

---

## Projet 1 — Le tenant Crossplane

### Objectif

Dans sa PR, **avant le merge**, un tenant déclare les signaux qu'il veut (metrics, logs, traces).
Une fois mergé, **les data sources Grafana correspondantes apparaissent**. C'est tout pour l'instant.

Contraintes :
- Tout est **namespacé**, façon Crossplane v2 : XR namespacé, managed resources `.m.`, pas de claims.
- Plus tard, l'API du tenant sera livrée comme **package OCI** (Configuration Crossplane), construit depuis GitHub et poussé sur un registry avec un tag.

### Contexte et hypothèses

- **Isolation des données : volontairement absente.** On n'utilise ni vmauth ni VMUser. Les data sources passent seulement les en-têtes `AccountID` / `ProjectID` à VictoriaMetrics, VictoriaLogs et VictoriaTraces. N'importe qui peut lire les données de n'importe qui, ce qui est acceptable pour un lab.
- **Convention :** l'ID du tenant (ex. `10001`) sert d'`AccountID`.
- **Briques Crossplane pressenties :**
  - **provider-grafana** pour les data sources. Vérifier qu'il expose des managed resources namespacées (`.m.`) dans la version installée.
  - **provider-kubernetes**, si besoin. Crossplane v2 sait déjà composer nativement n'importe quelle ressource Kubernetes : décider au cas par cas si le provider apporte quelque chose (cluster distant, mode observe-only…).
  - **CRDs des opérateurs Victoria** (ex. VMUser plus tard, si on ajoute de l'isolation), composables directement.

### Étapes

1. **Définir le contrat du tenant (l'API).** Lister ce que le tenant peut déclarer : son identifiant et ses signaux, limités aux valeurs autorisées. Ce schéma rejette une PR invalide avant qu'elle n'atteigne le cluster.
2. **Choisir qui crée les data sources Grafana.** provider-grafana (recommandé, plus formateur pour Crossplane) ou les CRs de grafana-operator composées directement.
3. **Fixer la convention multi-tenant.** Tenant `N` ⇒ `AccountID=N` dans les en-têtes de chaque data source.
4. **Installer la plomberie dans `platform/` :**
   - le ou les providers ;
   - les functions : un moteur de templating pour « si le signal est choisi, alors créer la data source », et **function-auto-ready**, indispensable pour que le Ready soit fiable, sans quoi l'overall time du projet 2 ne veut rien dire ;
   - la ProviderConfig **namespacée** et le secret d'accès à Grafana. Décider comment ce secret arrive par Flux (SOPS ?).
5. **Écrire la XRD** avec `scope: Namespaced`.
6. **Écrire la Composition en mode pipeline :** une data source par signal choisi, puis auto-ready. Poser un **label avec l'ID du tenant** sur tout ce qui est composé (le projet 2 en a besoin).
7. **Valider hors cluster :** `crossplane render` et `crossplane beta validate` en local, puis dans une GitHub Action sur chaque PR. C'est le « choisir avant le merge ».
8. **Onboarder le premier tenant dans `tenants/`.** Le namespace doit exister avant le XR : décider qui le crée.
9. **Jouer les scénarios :** ajout d'un tenant, retrait d'un signal (la data source doit disparaître), suppression du tenant.
10. **Ensuite seulement, le package OCI :** XRD, Composition et dépendances emballées dans une Configuration, construite et poussée sur GHCR par une GitHub Action avec un tag. `platform/` installe alors la Configuration par son tag. Dès le départ, ranger XRD et Composition dans un dossier à part, packageable tel quel.

---

## Projet 2 — Observabilité de l'onboarding

On ne l'attaque qu'une fois l'orchestration du projet 1 fonctionnelle.

### Ce qu'on veut voir, par tenant

- **Overall time.** Le temps réel entre la création du XR et son Ready. Il grimpe en direct tant que ce n'est pas fini (`now - création`), puis se fige. Ce n'est **pas** une somme des étapes : Crossplane crée les ressources en parallèle, donc la somme surestimerait la durée.
- **Timeline des étapes.** Chaque ressource composée avec son début, son Ready et sa durée. **L'étape la plus lente** est mise en avant, parce que c'est elle qui explique le total.
- **Ce qui bloque.** Les events Kubernetes `Warning` et les messages de condition (`Synced=False`, `Ready=False`), affichés sous la timeline.

Exemple attendu : « tenant-10001 : orchestration overall time : X s ».

### Pourquoi le panel actuel ne suffit pas

`crossplane_managed_resource_first_time_to_readiness_seconds` est agrégé par type de ressource (`gvk`). Il ne porte jamais l'identité du tenant.

### Approche retenue : mix métriques + logs

| Brique | Rôle |
|---|---|
| **kube-state-metrics** (CustomResourceState) | Exporte la date de création et la condition Ready (statut et date de transition) du XR et de chaque ressource composée, avec les labels tenant et ressource. Sert aux chiffres et à la timeline. Scrapé par le VM operator ou par le receiver prometheus OTel. |
| **Receiver OTel `k8sobjects`** (gateway) | Surveille les events Kubernetes liés aux ressources Crossplane et les envoie dans VictoriaLogs. Sert au « pourquoi ça bloque ». Nécessite d'étendre le ClusterRole du gateway. |

Les deux sources se croisent grâce au label tenant (et `crossplane.io/composite`).

Approches écartées :
- **Métriques seules :** les messages d'erreur ne tiennent pas dans des labels.
- **Logs seuls :** requêtes plus lourdes, et pas de belle state-timeline.
- **Une trace par tenant via un exporter maison :** la vue waterfall serait idéale, mais il y a du code à maintenir et ce n'est pas du direct. On la garde en bonus possible.

### Répartition

- **Côté Crossplane (fait par moi, guidé, sans code fourni) :** un Ready fiable (auto-ready), et le label tenant sur le XR et toutes les ressources composées.
- **Côté Claude :** le dashboard Grafana (panels et requêtes MetricsQL/LogsQL), la config de kube-state-metrics et celle du receiver `k8sobjects` / RBAC.

### Point ouvert

**Figer le premier passage à Ready.** `lastTransitionTime` reflète la *dernière* transition. Un tenant qui repasse en non-prêt puis redevient prêt verrait sa durée modifiée a posteriori. À trancher au démarrage du projet 2.
