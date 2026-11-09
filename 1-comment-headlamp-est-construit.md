# Comment Headlamp est construit

> Document d'architecture pour comprendre Headlamp — le projet upstream de ton fork `lighthouse` — avant de construire ta propre web UI Kubernetes.
>
> Le projet est un monorepo qui contient trois façons d'exécuter la même application : web in-cluster, desktop Electron, et mode développement local.

## Sommaire

1. [Vue d'ensemble du monorepo](#1-vue-densemble-du-monorepo)
2. [Le flux d'une requête](#2-le-flux-dune-requête)
3. [Le backend Go](#3-le-backend-go)
4. [Le frontend React](#4-le-frontend-react)
5. [L'application desktop Electron](#5-lapplication-desktop-electron)
6. [Les modes de déploiement](#6-les-modes-de-déploiement)
7. [Les principes de design](#7-les-principes-de-design)
8. [Ce qu'il faut retenir pour ton projet](#8-ce-quil-faut-retenir-pour-ton-projet)
9. [Ordre de lecture conseillé](#9-ordre-de-lecture-conseillé)

---

## 1. Vue d'ensemble du monorepo

| Dossier | Rôle | Stack principale |
|---|---|---|
| [`frontend/`](../frontend/) | L'interface utilisateur | React 18, TypeScript, Rsbuild, MUI, Redux Toolkit, TanStack Query, React Router v5 |
| [`backend/`](../backend/) | Serveur `headlamp-server` : SPA, auth, proxy Kubernetes, cache, WebSocket | Go, Gorilla mux, client-go, Helm |
| [`app/`](../app/) | Application desktop | Electron + TypeScript |
| [`plugins/`](../plugins/) | Outillage et exemples de plugins | `@kinvolk/headlamp-plugin` |
| [`charts/`](../charts/) | Déploiement Helm in-cluster | Helm |
| [`docs/`](../docs/) | Documentation développeur et utilisateur | Markdown / Docusaurus |

Points d'entrée :

- Frontend : [`frontend/src/index.tsx`](../frontend/src/index.tsx)
- Backend : [`backend/cmd/headlamp.go`](../backend/cmd/headlamp.go)
- Desktop : [`app/electron/main.ts`](../app/electron/main.ts)
- Documentation d'architecture officielle : [`docs/development/architecture.md`](../docs/development/architecture.md)

---

## 2. Le flux d'une requête

### Le trajet nominal

```
Navigateur (React)
   │  GET /clusters/mon-cluster/api/v1/namespaces/default/pods
   ▼
headlamp-server (Go)
   │  1. identifie le cluster dans le ContextStore
   │  2. applique l'authentification (kubeconfig, cookie, OIDC, service account)
   │  3. reverse-proxy HTTP vers l'API server du cluster
   ▼
Kubernetes API Server
```

La décision structurante du projet : **le navigateur ne parle jamais directement à l'API Kubernetes**. Tout passe par un backend qui joue le rôle de *BFF* (backend-for-frontend) et de proxy.

### Pourquoi ce choix

Ce choix résout quatre problèmes d'un coup :

1. **Authentification** — les credentials des clusters restent côté serveur. Le navigateur ne manipule qu'un cookie ou un token Headlamp.
2. **Multi-cluster** — l'URL contient le cluster (`/clusters/{nom}/...`) et le backend choisit le bon contexte.
3. **RBAC** — les refus `403` de l'API Kubernetes remontent naturellement à l'interface.
4. **Cache et temps réel** — le backend peut intercepter, mettre en cache, invalider et multiplexer les connexions.

### La route qui attrape tout

Dans [`backend/cmd/headlamp.go:2428`](../backend/cmd/headlamp.go#L2428) :

```go
router.PathPrefix("/clusters/{clusterName}/{api:.*}").Handler(
    auth.NewBackendTokenMiddleware(c.UseInCluster)(handler),
)
```

Le handler récupère le contexte du cluster puis appelle `Context.ProxyRequest` ([`backend/pkg/kubeconfig/kubeconfig.go:440`](../backend/pkg/kubeconfig/kubeconfig.go#L440)), qui utilise `httputil.NewSingleHostReverseProxy` vers l'URL du cluster ([`backend/pkg/kubeconfig/kubeconfig.go:484`](../backend/pkg/kubeconfig/kubeconfig.go#L484)).

Conséquence importante : le backend n'a pas besoin de connaître chaque type de ressource. Il relaie les chemins `/api/...` et `/apis/...` tels quels. C'est ce qui rend l'UI générique et compatible avec les CRDs.

### Côté frontend

Le préfixe `/clusters/{cluster}` est ajouté automatiquement dans [`frontend/src/lib/k8s/api/v2/fetch.ts:81`](../frontend/src/lib/k8s/api/v2/fetch.ts#L81) :

```ts
const urlParts = init.cluster ? ['clusters', init.cluster, url] : [url];
```

---

## 3. Le backend Go

### Les contextes de cluster

Au démarrage, le backend charge le kubeconfig et stocke chaque contexte dans un `ContextStore` ([`backend/pkg/kubeconfig/contextStore.go:35`](../backend/pkg/kubeconfig/contextStore.go#L35)).

C'est un cache en mémoire avec :

- TTL par contexte,
- listeners notifiés quand un contexte change ou expire,
- accès concurrent protégé (`sync.RWMutex`).

Chaque `Context` sait produire :

- une configuration client-go (`ClientConfig()`),
- un proxy HTTP (`SetupProxy` / `ProxyRequest`),
- son type d'authentification (`AuthType()`).

Les sources de contextes sont multiples : kubeconfig local, mode in-cluster, clusters dynamiques, inventaire de clusters.

### La configuration exposée au frontend

`GET /config` ([`backend/cmd/headlamp.go:2690`](../backend/cmd/headlamp.go#L2690)) renvoie :

- la liste des clusters,
- leurs métadonnées (source, namespace par défaut, extensions),
- les options activées (Helm, clusters dynamiques, etc.).

C'est le premier appel du frontend au chargement.

### L'authentification

Le backend gère plusieurs modes :

- token backend (`HEADLAMP_BACKEND_TOKEN`),
- token stocké en cookie,
- flow OIDC,
- service account in-cluster,
- clusters dynamiques où le kubeconfig est fourni par requête via les en-têtes `KUBECONFIG` et `X-HEADLAMP-USER-ID` (visibles dans `clusterFetch`).

Le middleware `auth.NewBackendTokenMiddleware` protège les routes selon le mode de déploiement. En mode in-cluster, l'utilisateur peut aussi être authentifié par un token OIDC et le backend transmet ce token à l'API server.

### Le cache

[`backend/pkg/k8cache/`](../backend/pkg/k8cache/) est un middleware de cache HTTP pour les requêtes GET Kubernetes.

- **Clé** : `apiGroup + kind + namespace + context` ([`backend/pkg/k8cache/cacheStore.go:206`](../backend/pkg/k8cache/cacheStore.go#L206)).
- **TTL** : 10 minutes ([`backend/pkg/k8cache/cacheStore.go:323`](../backend/pkg/k8cache/cacheStore.go#L323)).
- **Invalidation** : les écritures (POST/PUT/PATCH/DELETE) suppriment les entrées concernées ([`backend/pkg/k8cache/cacheInvalidation.go:62`](../backend/pkg/k8cache/cacheInvalidation.go#L62)).
- **Exclusion** : les WebSockets ne passent jamais par le cache, pour garantir le temps réel.
- **Traçabilité** : une réponse servie depuis le cache porte l'en-tête `X-HEADLAMP-CACHE: true`.

Le cache utilise aussi des informers Kubernetes pour surveiller un ensemble de ressources importantes et invalider les entrées quand elles changent.

### Le temps réel et le multiplexeur WebSocket

Les navigateurs limitent le nombre de connexions WebSocket par origine (souvent six). Headlamp ouvre donc :

- **une seule WebSocket** entre le navigateur et le backend : `/wsMultiplexer`,
- **autant de WebSockets que nécessaire** entre le backend et les API servers.

Chaque message est routé par un triplet `clusterId / path / query`. L'implémentation est dans [`backend/cmd/multiplexer.go`](../backend/cmd/multiplexer.go).

Le frontend fait de même avec un gestionnaire singleton ([`frontend/src/lib/k8s/api/v2/multiplexer.ts:30`](../frontend/src/lib/k8s/api/v2/multiplexer.ts#L30)) qui :

- garde la liste des souscriptions actives,
- se reconnecte automatiquement,
- se réabonne après une reconnexion,
- nettoie les souscriptions inutilisées.

### Les plugins côté serveur

Les routes `/plugins/`, `/user-plugins/` et `/static-plugins/` ([`backend/cmd/headlamp.go:571`](../backend/cmd/headlamp.go#L571)) servent les bundles JavaScript des plugins.

Le backend ne les exécute pas : il les stocke et les sert. C'est le navigateur qui les charge et les exécute.

### Les services annexes

Le même serveur Go expose aussi :

- le proxy de services in-cluster (`/clusters/{cluster}/serviceproxy/...`),
- Helm (`/clusters/{cluster}/helm/...`),
- le drain de nœuds,
- le port-forward,
- les métriques Prometheus,
- le tracing OpenTelemetry.

C'est un serveur unique qui agrège toutes les capacités nécessaires à l'UI.

---

## 4. Le frontend React

### Le bootstrap

1. [`frontend/src/index.tsx`](../frontend/src/index.tsx) monte `<App />`.
2. [`frontend/src/App.tsx`](../frontend/src/App.tsx) empile les providers :
   - Redux,
   - TanStack Query,
   - thème MUI,
   - i18n.
3. [`AppContainer`](../frontend/src/components/App/AppContainer.tsx) choisit `BrowserRouter` (web) ou `HashRouter` (Electron), puis monte le layout.

### Deux états bien séparés

C'est une leçon importante de l'architecture :

- **Redux Toolkit** pour l'état applicatif : thème, préférences, routes enregistrées par les plugins, filtres, projets, raccourcis.
- **TanStack Query** pour l'état serveur : les ressources Kubernetes, avec cache, invalidation, refetch et clés de requête.

Headlamp ne met pas les objets Kubernetes dans Redux. Il les traite comme des données serveur, ce qui évite de réinventer un cache et de synchroniser manuellement deux sources de vérité.

### Le modèle `KubeObject`

[`frontend/src/lib/k8s/KubeObject.ts:55`](../frontend/src/lib/k8s/KubeObject.ts#L55) est la classe de base.

Chaque ressource a son fichier (`pod.ts`, `deployment.ts`, `service.ts`, etc.) qui déclare :

- `kind`,
- `apiVersion`,
- `apiName` (le pluriel utilisé dans l'API),
- `isNamespaced`,
- les colonnes et actions spécifiques.

La classe fournit les opérations génériques : `list`, `get`, `create`, `delete`, `patch`, ainsi que les contrôles d'autorisation. C'est ce qui permet d'avoir une seule table générique pour toutes les ressources.

### Les hooks

[`frontend/src/lib/k8s/api/v2/hooks.ts:108`](../frontend/src/lib/k8s/api/v2/hooks.ts#L108) expose :

- `useKubeObject` pour une ressource unique,
- `useKubeObjectList` pour une liste.

Ils sont construits sur React Query. La clé de requête contient le cluster, l'endpoint, le namespace, le nom et les paramètres. C'est ce qui rend le cache et l'invalidation corrects en multi-cluster.

### Le routage multi-cluster

Les routes sont de la forme `/c/:cluster/...` ([`frontend/src/lib/cluster.ts:28`](../frontend/src/lib/cluster.ts#L28)).

Le cluster courant est extrait de l'URL. Il peut contenir plusieurs clusters sélectionnés séparés par `+` pour les vues agrégées. Le premier cluster de la liste est généralement celui de la page de détail.

### Les plugins côté frontend

[`frontend/src/plugin/registry.tsx`](../frontend/src/plugin/registry.tsx) expose un vocabulaire d'extension très riche :

- `registerRoute`
- `registerSidebarEntry`
- `registerAppBarAction`
- `registerDetailsViewSection`
- `registerResourceTableColumnsProcessor`
- `registerAppTheme`
- `registerKubeObjectGlance`
- `registerHeadlampEventCallback`
- etc.

Les plugins sont des bundles JavaScript qui appellent ces fonctions au chargement. [`frontend/src/plugin/lib.ts`](../frontend/src/plugin/lib.ts) est la surface publique destinée aux plugins.

Un plugin peut donc ajouter une page, une colonne, une action ou un panneau sans forker le cœur.

---

## 5. L'application desktop Electron

[`app/`](../app/) contient tout ce qui concerne l'application desktop.

- Entrée : [`app/electron/main.ts`](../app/electron/main.ts).
- Electron démarre le backend Go et charge le frontend.
- Le frontend détecte le mode Electron et utilise `HashRouter` ainsi que l'API `window.desktopApi` exposée par le preload.
- L'application peut installer des plugins localement dans `~/.config/Headlamp/plugins`.

Le desktop n'est pas un composant séparé de l'architecture : c'est un **hôte** qui embarque le même backend et le même frontend.

---

## 6. Les modes de déploiement

| Mode | Fonctionnement |
|---|---|
| **In-cluster** | Le backend et le frontend tournent dans un pod. Un Ingress expose le service. L'auth se fait par token, OIDC ou service account. |
| **Desktop** | Electron embarque le binaire Go et les fichiers statiques du frontend. Le kubeconfig local est utilisé. |
| **Développement local** | `npm start` lance le backend Go et le serveur de développement frontend en parallèle. |

Les manifests et le chart Helm sont dans [`charts/`](../charts/), le Dockerfile à la racine du dépôt.

---

## 7. Les principes de design

Dans [`docs/development/architecture.md`](../docs/development/architecture.md) :

- **Extensibilité** — tout ce qui peut être un plugin doit pouvoir l'être.
- **Multi-cluster** — plusieurs clusters en même temps, dans la même UI.
- **Sécurité** — l'UI suit les règles RBAC de Kubernetes ; elle n'invente pas ses permissions.
- **Accessibilité** — l'interface doit être utilisable par tout le monde.
- **Temps réel** — l'état affiché doit suivre le cluster.
- **Cohérence** — l'UI doit toujours montrer l'état courant du cluster.
- **UI adaptative** — si l'utilisateur n'a pas le droit d'éditer, il ne voit qu'une action de consultation.

---

## 8. Ce qu'il faut retenir pour ton projet

1. **Un backend proxy mince est non négociable** si tu veux gérer l'auth, le multi-cluster et le RBAC proprement. Le navigateur ne doit pas détenir les credentials des clusters.
2. **Une couche générique par GVR** (Group/Version/Resource) vaut mieux qu'une page par type de ressource. Tu listes, détailles et édites n'importe quelle ressource, y compris les CRDs, avec le même code.
3. **Sépare l'état serveur de l'état UI** : React Query pour les ressources, un store léger pour le reste.
4. **Le watcher est la partie qui change tout** : dès que tu passes en temps réel, tu dois gérer reconnexion, `resourceVersion`, annulation et fuites de connexions.
5. **Les plugins sont une couche avancée.** Headlamp peut se le permettre parce que son cœur est stable. Pour un side project, commence sans plugins : ton équivalent, c'est une bonne couche de ressources génériques.
6. **Commence en lecture seule.** Les écritures, l'exec et les logs sont des transports différents, avec leurs propres erreurs (403, conflits, timeouts).

---

## 9. Ordre de lecture conseillé

Pour t'imprégner du code sans te noyer :

1. [`docs/development/architecture.md`](../docs/development/architecture.md) — la vue d'ensemble officielle.
2. [`backend/cmd/headlamp.go`](../backend/cmd/headlamp.go) — le câblage des routes, en particulier `handleClusterRequests` et `handleClusterAPI`.
3. [`backend/pkg/kubeconfig/kubeconfig.go`](../backend/pkg/kubeconfig/kubeconfig.go) — les contextes et le proxy.
4. [`backend/pkg/k8cache/`](../backend/pkg/k8cache/) — le cache et son invalidation.
5. [`backend/cmd/multiplexer.go`](../backend/cmd/multiplexer.go) — le multiplexeur WebSocket.
6. [`frontend/src/lib/k8s/KubeObject.ts`](../frontend/src/lib/k8s/KubeObject.ts) — le modèle de ressources.
7. [`frontend/src/lib/k8s/api/v2/`](../frontend/src/lib/k8s/api/v2/) — fetch, hooks React Query, WebSocket.
8. [`frontend/src/plugin/registry.tsx`](../frontend/src/plugin/registry.tsx) — l'API de plugins, en dernier.

Le guide de construction de ta propre UI est dans [`todo/2-guide-detaille.md`](2-guide-detaille.md), et la feuille de route suivie est dans [`ROADMAP.md`](../ROADMAP.md).
