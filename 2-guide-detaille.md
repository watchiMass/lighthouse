# Guide détaillé — construire ta propre web UI Kubernetes

> Objectif : une UI personnelle pour tes clusters, utilisable en local d'abord, puis déployable pour une équipe.
>
> Ce guide est le compagnon détaillé de [`ROADMAP.md`](../ROADMAP.md). La roadmap sert à cocher les étapes ; ce document explique **quoi faire, dans quel ordre, et pourquoi**.
>
> Approche choisie : **construire depuis zéro**, avec Headlamp comme modèle d'architecture. Voir [`todo/1-comment-headlamp-est-construit.md`](1-comment-headlamp-est-construit.md) pour l'analyse du projet de référence.

## Sommaire

1. [L'architecture cible](#1-larchitecture-cible)
2. [La stack et pourquoi](#2-la-stack-et-pourquoi)
3. [Les prérequis](#3-les-prérequis)
4. [Le squelette du projet](#4-le-squelette-du-projet)
5. [Les milestones en détail](#5-les-milestones-en-détail)
6. [La stratégie de test](#6-la-stratégie-de-test)
7. [La checklist sécurité](#7-la-checklist-sécurité)
8. [Le passage à l'équipe](#8-le-passage-à-léquipe)
9. [Les projets comparables](#9-les-projets-comparables)
10. [Le premier pas](#10-le-premier-pas)

---

## 1. L'architecture cible

```
┌──────────────────────────────┐
│  Navigateur                  │
│  React + TanStack Query      │
│  (aucun credential K8s)      │
└──────────────┬───────────────┘
               │ HTTP / WebSocket
               ▼
┌──────────────────────────────┐
│  BFF (Node + Express)        │
│  - charge le kubeconfig      │
│  - choisit le contexte       │
│  - parle à l'API K8s         │
│  - streame watch/logs/exec   │
└──────────────┬───────────────┘
               │ HTTPS
               ▼
┌──────────────────────────────┐
│  Kubernetes API Server       │
│  + RBAC + admission          │
└──────────────────────────────┘
```

Les trois invariants à garder en tête :

1. **Le navigateur ne détient jamais les credentials des clusters.** Le BFF est le seul à parler à l'API Kubernetes.
2. **Le frontend ne connaît pas les types de ressources.** Il manipule des objets génériques identifiés par leur GVR (Group/Version/Resource).
3. **Le cluster est explicite dans l'URL.** Chaque requête sait à quel cluster elle s'adresse.

---

## 2. La stack et pourquoi

| Couche | Choix | Justification |
|---|---|---|
| Frontend | React + TypeScript + Vite | Même modèle que Headlamp, outillage minimal au départ. |
| UI | MUI | Composants de table, dialogue, formulaire prêts à l'emploi ; tu peux t'inspirer de Headlamp. |
| Données serveur | TanStack Query | Cache, refetch, invalidation, clés de requête : le bon outil pour des ressources K8s. |
| Backend | Node.js + TypeScript + Express | Un seul langage pour tout le projet. La lib `@kubernetes/client-node` gère kubeconfig, auth, watch, logs et exec. |
| Tests | Vitest + Supertest | Rapides, suffisants pour tester le BFF et les composants. |
| Cluster de test | kind ou minikube | Permet de casser et recréer un cluster sans risque. |

### Le compromis à assumer

Le BFF Node est le chemin le plus court pour apprendre. Si tu veux plus tard :

- un **binaire unique**,
- une parité maximale avec Headlamp,
- de meilleures performances sur de gros clusters,

alors remplace le BFF par **Go + client-go**. C'est le choix de Headlamp, Radar et Kite. Mais ne fais pas ce changement avant d'avoir un milestone 5 qui fonctionne : tu saurais alors exactement ce que le BFF doit faire.

### Ce que tu n'installes pas au début

- Pas de base de données.
- Pas de système de plugins.
- Pas d'Electron.
- Pas d'i18n.
- Pas de design system maison.
- Pas de CI/CD tant que le squelette ne tourne pas.

---

## 3. Les prérequis

- **Node.js 22+** et npm.
- **kubectl** configuré sur au moins un cluster.
- Un cluster de test local recommande : **kind** ou **minikube**.
- Un namespace de test, par exemple `demo`.
- De quoi lire le code Go et TypeScript de Headlamp sans le modifier.

Vérifications rapides :

```bash
node --version
kubectl version --client
kubectl config get-contexts
kubectl get pods -A
```

Si `kubectl get pods -A` fonctionne, ton kubeconfig et ton auth sont bons. C'est le prérequis le plus important : ton BFF utilisera le même kubeconfig.

---

## 4. Le squelette du projet

Suggestion de disposition :

```
kubeui/
├── web/                     # React + Vite
│   ├── src/
│   │   ├── api/             # appels au BFF
│   │   ├── components/
│   │   ├── pages/
│   │   └── main.tsx
│   └── vite.config.ts       # proxy /api -> localhost:3001
├── server/                  # BFF Node + Express
│   ├── src/
│   │   ├── index.ts
│   │   ├── kubeconfig.ts    # chargement et contexte courant
│   │   ├── routes/
│   │   └── k8s/             # découverte, listes, watch
│   └── package.json
├── ROADMAP.md
└── README.md
```

Deux processus en développement :

- le serveur Express sur `http://localhost:3001`,
- Vite sur `http://localhost:5173`, avec un proxy `/api` vers Express.

Ainsi, le navigateur ne fait que des appels same-origin à Vite, et il n'y a aucun problème de CORS.

---

## 5. Les milestones en détail

### Milestone 1 — Squelette qui lit les pods d'un cluster

**Objectif** : prouver que la chaîne `React → BFF → kubeconfig → API Kubernetes` fonctionne.

**Tâches**

1. Générer l'app React avec Vite.
2. Créer le serveur Express en TypeScript.
3. Charger le kubeconfig avec `KubeConfig.loadFromDefault()`.
4. Exposer `GET /api/pods` qui liste les pods de tous les namespaces.
5. Configurer le proxy Vite `/api` vers `http://localhost:3001`.
6. Afficher les pods dans un tableau avec les états `loading`, `error` et `empty`.

**Critère de done**

`npm run dev` affiche les pods du contexte courant, et tu peux les comparer à `kubectl get pods -A`.

**Pièges**

- **Auth exec** : si ton cluster utilise un provider cloud (`aws`, `gke`, `az`), le kubeconfig peut invoquer un binaire d'authentification. `@kubernetes/client-node` sait le faire, mais vérifie que le binaire est dans le `PATH`.
- **TLS** : ne désactive jamais la vérification TLS pour « faire marcher » un cluster de test ; utilise plutôt le bon `certificate-authority-data`.
- **Forme de la réponse** : selon la version de `@kubernetes/client-node`, la méthode renvoie directement l'objet ou `{ body }`. Vérifie la version installée.
- **Namespace** : `listPodForAllNamespaces` nécessite les droits RBAC correspondants. En cas de 403, teste d'abord dans un namespace.
- **Ne logue jamais le kubeconfig**, il contient des tokens.

**À lire dans Headlamp**

- [`frontend/src/lib/k8s/api/v2/fetch.ts`](../frontend/src/lib/k8s/api/v2/fetch.ts) — le préfixe `/clusters/{cluster}`.
- [`backend/cmd/headlamp.go`](../backend/cmd/headlamp.go) — la route proxy.

---

### Milestone 2 — Couche de ressources générique (GVR)

**Objectif** : arrêter de coder une page par type de ressource.

**Le concept**

Kubernetes expose deux familles de chemins :

| Type | Chemin |
|---|---|
| Groupe core (`v1`) | `/api/v1/...` |
| Groupe nommé | `/apis/{group}/{version}/...` |
| Ressource namespaced | `.../namespaces/{namespace}/{resource}` |
| Ressource cluster-scoped | `.../{resource}` |

Un GVR est le triplet `group`, `version`, `resource` (le pluriel, par exemple `pods`, `deployments`).

**Tâches**

1. Interroger la découverte : `/api`, `/apis`, puis `/api/v1` et `/apis/{group}/{version}`.
2. Construire un catalogue de ressources : `group`, `version`, `resource`, `kind`, `namespaced`, `verbs`.
3. Exposer un endpoint générique :
   `GET /api/clusters/:cluster/resources?group=&version=&resource=&namespace=`
4. Lister avec `CustomObjectsApi` ou `KubernetesObjectApi` selon le cas.
5. Ajouter un sélecteur de ressource dans l'UI et une table générique.
6. Afficher au minimum `name`, `namespace` et `age` ; le reste en JSON repliable.

**Critère de done**

Tu passes de `pods` à `deployments`, `services` et `configmaps` via un menu, sans changer le code de la page.

**Pièges**

- **Groupe core** : `group` est vide ; le chemin est `/api/v1`, pas `/apis//v1`.
- **Sous-ressources** : `pods/log`, `deployments/scale`, etc. Filtre les entrées contenant `/`.
- **Découverte partielle** : certains API groups agrégés peuvent échouer ; une erreur sur un groupe ne doit pas casser tout le catalogue.
- **Pagination** : les grosses listes renvoient `metadata.continue` ; prévois `limit` et `continue`.
- **Verbes** : utilise `verbs` pour savoir si tu peux lister, créer, supprimer.
- **CRDs** : elles apparaissent dans la découverte comme les autres ressources ; c'est tout l'intérêt de la couche générique.

**À lire dans Headlamp**

- [`frontend/src/lib/k8s/KubeObject.ts`](../frontend/src/lib/k8s/KubeObject.ts)
- [`frontend/src/lib/k8s/api/v2/KubeObjectEndpoint.ts`](../frontend/src/lib/k8s/api/v2/KubeObjectEndpoint.ts)
- [`frontend/src/lib/k8s/api/v2/apiDiscovery.tsx`](../frontend/src/lib/k8s/api/v2/apiDiscovery.tsx)

---

### Milestone 3 — Multi-contextes et URL par cluster

**Objectif** : gérer plusieurs clusters proprement avant que le reste du code ne fige des hypothèses mono-cluster.

**Tâches**

1. Lister les contextes avec `kc.getContexts()`.
2. Exposer `GET /api/clusters`.
3. Ajouter la route frontend `/c/:cluster/...`.
4. Ajouter un sélecteur de cluster dans l'en-tête.
5. Transmettre le cluster à chaque requête, soit dans le chemin, soit dans un paramètre validé.
6. Côté BFF, charger le contexte demandé pour chaque requête.

**Le piège principal : la mutabilité du `KubeConfig`**

`kc.setCurrentContext(name)` modifie un objet global. Si deux requêtes arrivent en parallèle pour deux clusters différents, elles peuvent se marcher dessus.

Solutions possibles, de la plus simple à la plus robuste :

- recharger le kubeconfig et sélectionner le contexte à chaque requête,
- garder un cache de `KubeConfig` par cluster,
- utiliser un client dédié par cluster.

Pour un side project, le cache par cluster suffit.

**Critère de done**

Deux contextes différents dans `kubectl config get-contexts` affichent des ressources différentes dans l'UI, sans redémarrer le serveur.

**Pièges**

- **Noms de contextes** : ils peuvent contenir des caractères spéciaux (`/`, `+`, espaces). Encode-les dans l'URL.
- **Namespace par défaut** : chaque contexte peut avoir un namespace par défaut différent.
- **Auth par contexte** : ne réutilise pas le client d'un cluster pour un autre.
- **Clusters dynamiques** : plus tard, tu pourras accepter un kubeconfig fourni par l'utilisateur ; ne le fais pas avant d'avoir sécurisé le reste.

**À lire dans Headlamp**

- [`backend/pkg/kubeconfig/contextStore.go`](../backend/pkg/kubeconfig/contextStore.go)
- [`frontend/src/lib/cluster.ts`](../frontend/src/lib/cluster.ts)
- [`backend/cmd/headlamp.go`](../backend/cmd/headlamp.go) — la route `/clusters/{clusterName}/...`.

---

### Milestone 4 — Vue détail et YAML en lecture seule

**Objectif** : inspecter une ressource en profondeur sans encore la modifier.

**Tâches**

1. Ajouter une route de détail : `/c/:cluster/:resource/:namespace/:name`.
2. Récupérer l'objet unique via le même endpoint générique.
3. Afficher les métadonnées : nom, namespace, labels, annotations, age, propriétaires.
4. Afficher les conditions et le statut quand ils existent.
5. Afficher le YAML avec `js-yaml`, en lecture seule.
6. Ajouter un bouton « copier » et un bouton « télécharger ».

**Critère de done**

Cliquer sur un pod affiche son YAML et ses métadonnées, sans passer par `kubectl`.

**Pièges**

- **Secrets** : ne montre jamais les valeurs de `data` en clair par défaut ; masque-les derrière un bouton « révéler ».
- **`managedFields`** : le YAML est souvent énorme à cause de ce champ ; propose de le masquer.
- **Objets volumineux** : un CRD peut produire un YAML très long ; prévois un éditeur ou un viewer scrollable.
- **Champs absents** : ne suppose pas que `status` ou `conditions` existent.
- **Ordre des clés** : `js-yaml` ne préserve pas l'ordre ; ce n'est pas grave en lecture seule.

**À lire dans Headlamp**

- [`backend/pkg/kubeconfig/kubeconfig.go`](../backend/pkg/kubeconfig/kubeconfig.go) — le proxy de détail.
- [`frontend/src/lib/k8s/KubeObject.ts`](../frontend/src/lib/k8s/KubeObject.ts) — `detailsRoute`.
- Les composants `DetailsViewSection` dans [`frontend/src/components/`](../frontend/src/components/).

---

### Milestone 5 — Temps réel avec `watch`

**Objectif** : la liste se met à jour toute seule, comme dans Headlamp.

**Le concept**

L'API Kubernetes expose un paramètre `watch=1`. Le serveur renvoie un flux d'événements :

- `ADDED`
- `MODIFIED`
- `DELETED`
- `BOOKMARK`
- `ERROR`

Chaque événement contient l'objet et une `resourceVersion`. Tu peux reprendre un watch à partir d'une `resourceVersion` connue.

**Tâches**

1. Côté BFF, utiliser `new k8s.Watch(kc)` sur le chemin de la ressource.
2. Transmettre les événements au navigateur, via SSE (plus simple) ou WebSocket.
3. Côté frontend, mettre à jour le cache TanStack Query avec `queryClient.setQueryData`.
4. Gérer la reconnexion avec backoff.
5. Sur une erreur `410 Gone` (resourceVersion trop ancienne), refaire une liste complète puis reprendre le watch.
6. Nettoyer le watch quand l'utilisateur quitte la page.

**Critère de done**

`kubectl delete pod <nom>` fait disparaître le pod de l'UI en quelques secondes, sans rafraîchir la page.

**Pièges**

- **Timeout serveur** : les watches Kubernetes expirent au bout de quelques minutes ; il faut se reconnecter.
- **`resourceVersion`** : ne mélange pas les versions entre plusieurs listes.
- **`BOOKMARK`** : sert à maintenir la `resourceVersion` à jour sans envoyer d'objet complet.
- **Fuite de connexions** : un watch oublié continue de consommer des ressources côté API server.
- **Backpressure** : si le frontend est lent, le BFF doit pouvoir abandonner ou bufferiser proprement.
- **Limite navigateur** : si tu ouvres un WebSocket par liste, tu vas saturer. C'est exactement le problème que Headlamp résout avec son multiplexeur ; pour un side project, commence par **un seul WebSocket par onglet de navigateur** et route les souscriptions par un identifiant.

**À lire dans Headlamp**

- [`frontend/src/lib/k8s/api/v2/webSocket.ts`](../frontend/src/lib/k8s/api/v2/webSocket.ts)
- [`frontend/src/lib/k8s/api/v2/multiplexer.ts`](../frontend/src/lib/k8s/api/v2/multiplexer.ts)
- [`frontend/src/lib/k8s/api/v2/useKubeObjectList.ts`](../frontend/src/lib/k8s/api/v2/useKubeObjectList.ts)
- [`backend/cmd/multiplexer.go`](../backend/cmd/multiplexer.go)

---

### Milestone 6 — Écriture : créer, éditer, supprimer

**Objectif** : agir sur le cluster, pas seulement l'observer.

**Tâches**

1. Ajouter un éditeur YAML (Monaco, ou un simple textarea au début).
2. Ajouter les actions `create`, `replace` / `patch`, `delete`.
3. Ajouter une confirmation avant toute suppression.
4. Afficher les erreurs de l'API de façon lisible :
   - `403` : droits insuffisants,
   - `409` : conflit de `resourceVersion`,
   - `422` : objet invalide.
5. Rafraîchir la liste après une écriture réussie.

**Critère de done**

Tu modifies un ConfigMap dans l'UI et tu supprimes un pod, puis `kubectl` confirme le changement.

**Pièges**

- **`resourceVersion`** : si tu fais un `replace` avec une version périmée, l'API renvoie `409`. C'est une protection, pas un bug.
- **Champs immutables** : certains champs ne peuvent pas être modifiés ; l'API renverra `422`.
- **Serveur-side apply** : plus puissant, mais plus complexe. Commence par `replace`/`patch`, puis envisage `apply` plus tard.
- **Suppression** : `propagationPolicy` (Foreground/Background/Orphan) change le comportement des dépendants.
- **Sécurité** : une confirmation ne remplace pas le RBAC. Le backend doit envoyer la requête avec les droits de l'utilisateur, pas avec un super-compte.

**À lire dans Headlamp**

- [`frontend/src/lib/k8s/KubeObject.ts`](../frontend/src/lib/k8s/KubeObject.ts) — les méthodes `create`, `delete`, `patch`.
- Les actions de détail dans [`frontend/src/components/`](../frontend/src/components/).

---

### Milestone 7 — Logs et exec

**Objectif** : diagnostiquer un pod sans quitter l'UI.

**Tâches**

1. **Logs** : exposer un endpoint qui streame `GET /api/v1/namespaces/{ns}/pods/{pod}/log?follow=true`.
2. **Frontend** : afficher le flux dans un terminal (`xterm.js`) ou un simple panneau scrollable.
3. **Exec** : utiliser `new k8s.Exec(kc)` côté BFF, et un WebSocket entre le navigateur et le BFF.
4. Gérer la sélection du conteneur (y compris les init containers).
5. Gérer le redimensionnement du terminal et l'envoi de `stdin`.
6. Gérer la reconnexion et la fermeture propre.

**Critère de done**

Tu ouvres les logs d'un pod, puis un shell dans un conteneur, depuis l'UI.

**Pièges**

- **Multi-conteneurs** : un pod peut avoir plusieurs conteneurs ; il faut choisir.
- **Protocole exec** : ce n'est pas du texte brut ; les frames ont un type (stdin, stdout, stderr, resize).
- **Backpressure** : un pod bavard peut saturer le navigateur ; limite la taille du buffer.
- **Timeouts** : les connexions exec peuvent être coupées par un proxy ou un ingress ; prévois un message clair.
- **Sécurité** : `pods/exec` est une permission très sensible ; ne l'accorde pas à un compte de service trop large.

**À lire dans Headlamp**

- [`frontend/src/lib/k8s/useTerminalStream.ts`](../frontend/src/lib/k8s/useTerminalStream.ts)
- [`backend/pkg/portforward/`](../backend/pkg/portforward/)
- [`backend/pkg/serviceproxy/`](../backend/pkg/serviceproxy/)

---

### Milestone 8 — Premier déploiement « équipe »

**Objectif** : passer d'un outil local à un service accessible à plusieurs personnes.

**Tâches**

1. **Image Docker** multi-stage : build frontend, build BFF, image finale légère.
2. **Manifests Kubernetes** ou chart Helm : Deployment, Service, Ingress, ConfigMap.
3. **ServiceAccount** dédié avec un RBAC minimal.
4. **Auth** : token, OIDC, ou délégation au token utilisateur.
5. **Isolation** : chaque utilisateur ne doit voir que ce que son RBAC autorise.
6. **Santé** : `readinessProbe` et `livenessProbe`.
7. **Observabilité** : logs structurés, métriques de base.
8. **TLS** et support WebSocket dans l'Ingress.

**Critère de done**

Un coéquipier ouvre l'URL déployée, s'authentifie, et ne voit que les ressources autorisées par son RBAC.

**Pièges**

- **ServiceAccount trop puissant** : si le BFF utilise un SA `cluster-admin`, tous les utilisateurs deviennent admin. C'est la faute la plus grave possible.
- **Isolation multi-utilisateur** : deux modèles possibles :
  - **impersonation** : le backend agit au nom de l'utilisateur,
  - **token utilisateur** : le frontend envoie le token de l'utilisateur, le backend le relaie.
  Le second est plus simple à comprendre ; le premier est plus flexible.
- **OIDC** : les URLs de redirection doivent correspondre exactement à ce qui est déclaré chez l'IdP.
- **Cookies** : `HttpOnly`, `Secure`, `SameSite=Lax` ou `Strict` selon le flux.
- **CSRF** : si tu utilises des cookies pour l'auth, protège les écritures.
- **Secrets** : ne les logue jamais, même en debug.
- **Audit** : journalise les actions d'écriture (qui, quoi, quand).

**À lire dans Headlamp**

- [`backend/pkg/auth/`](../backend/pkg/auth/)
- [`Dockerfile`](../Dockerfile)
- [`charts/`](../charts/)
- [`docs/installation/in-cluster/index.md`](../docs/installation/in-cluster/index.md)

---

## 6. La stratégie de test

L'objectif n'est pas d'avoir 100 % de couverture, mais de tester ce qui casse silencieusement.

| Niveau | Quoi tester | Outil |
|---|---|---|
| Unitaire | Construction d'URL, encodage des clusters, clés de cache | Vitest |
| Intégration BFF | Listes, détails, erreurs 403/404, avec un vrai cluster kind | Supertest + un cluster kind |
| Composants | Tableau, états loading/error/empty, formulaire YAML | Testing Library |
| E2E (plus tard) | Scénario complet : voir un pod, ouvrir le détail, supprimer | Playwright |

Le conseil de Headlamp est de **préférer les vraies implémentations aux mocks**. Pour le BFF, un cluster kind jetable est plus fiable qu'un mock de l'API Kubernetes : il valide aussi ton auth et ton RBAC.

Exemple de boucle :

```bash
kind create cluster --name kubeui-test
kubectl create namespace demo
kubectl run demo-pod --image=nginx -n demo
npm test
kind delete cluster --name kubeui-test
```

---

## 7. La checklist sécurité

À vérifier avant chaque milestone, puis à revoir avant le déploiement :

- [ ] Le kubeconfig et les tokens ne sont **jamais** envoyés au navigateur.
- [ ] Le BFF valide le nom du cluster et ne permet pas d'URL d'API server arbitraire (risque SSRF).
- [ ] Le backend applique l'autorisation ; l'UI n'est qu'un confort.
- [ ] Les Secrets sont masqués par défaut et jamais logués.
- [ ] Les cookies d'auth sont `HttpOnly`, `Secure`, et protégés contre le CSRF.
- [ ] La vérification TLS est activée.
- [ ] Le ServiceAccount in-cluster est au moindre privilège.
- [ ] Les actions destructrices demandent une confirmation.
- [ ] Les erreurs de l'API ne révèlent pas d'informations sensibles.
- [ ] Les dépendances sont à jour (au moins avant le déploiement).

---

## 8. Le passage à l'équipe

Le milestone 8 est un changement de nature, pas une simple étape. Les questions à trancher :

1. **Qui authentifie ?** Ton app, ou l'IdP de l'organisation ?
2. **Quel modèle d'identité ?** Un token utilisateur relayé, ou de l'impersonation ?
3. **Quel périmètre ?** Un namespace, un cluster, plusieurs clusters ?
4. **Quel niveau de confiance ?** Lecture seule pour tous, écriture pour certains ?
5. **Où tourne l'app ?** In-cluster, ou hors cluster avec accès réseau aux API servers ?

Tant que ces questions ne sont pas tranchées, ne construis pas le milestone 8. Un déploiement multi-utilisateur approximatif est plus dangereux qu'un outil local.

---

## 9. Les projets comparables

| Projet | Ce qu'il faut y étudier |
|---|---|
| [Headlamp](https://github.com/kubernetes-sigs/headlamp) | L'architecture complète : proxy Go, `KubeObject`, React Query + watch, plugins. C'est ta référence principale. |
| [Radar](https://github.com/skyhook-io/radar) | Un binaire Go unique avec frontend embarqué, informers et SSE. Plus simple que Headlamp. |
| [Kite](https://github.com/zxh326/kite) | Dashboard multi-cluster Go + React 19. Bonne checklist de fonctionnalités (Helm, Prometheus, RBAC, OIDC). |
| [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) | Archivé depuis janvier 2026. À regarder pour l'UX historique, pas comme base de code. |

---

## 10. Le premier pas

```bash
mkdir -p ~/Works/kubeui && cd ~/Works/kubeui
npm create vite@latest web -- --template react-ts
mkdir server && cd server
npm init -y
npm i express @kubernetes/client-node
npm i -D typescript tsx @types/express @types/node
```

Ensuite, crée `server/src/index.ts`. Le concept à valider tient en quelques lignes :

```ts
import express from 'express';
import * as k8s from '@kubernetes/client-node';

const kc = new k8s.KubeConfig();
kc.loadFromDefault(); // ~/.kube/config
const core = kc.makeApiClient(k8s.CoreV1Api);

const app = express();
app.get('/api/pods', async (_req, res) => {
  res.json(await core.listPodForAllNamespaces());
});
app.listen(3001);
```

Puis configure le proxy Vite `/api` → `http://localhost:3001` dans `web/vite.config.ts`.

Si ta version de `@kubernetes/client-node` renvoie encore `{ body }`, utilise `result.body` : la forme a changé selon les versions.

Le but du milestone 1 n'est pas d'avoir une belle UI. C'est de prouver que la chaîne `React → BFF → kubeconfig → API K8s` fonctionne. Le reste de la roadmap construit sur cette fondation.

---

Quand le milestone 1 est coché dans [`ROADMAP.md`](../ROADMAP.md), passe au milestone 2 et relis la section GVR ci-dessus. Si tu bloques, reviens avec le code et l'erreur : on débloque le concept, pas le projet à ta place.
