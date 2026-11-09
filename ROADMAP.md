# KubeUI — Roadmap
Goal: Une web UI personnelle pour mes clusters Kubernetes, locale d'abord puis déployable pour une équipe.
Stack: React + TypeScript + Vite (frontend) ; Node.js + TypeScript + Express + `@kubernetes/client-node` (BFF) ; TanStack Query ; MUI ; Vitest.

- [ ] 1. Squelette qui lit les pods d'un cluster
  - Do: Une page React qui liste les pods via un petit serveur Express qui charge ton kubeconfig et appelle l'API Kubernetes.
  - Done when: `npm run dev` affiche les pods du contexte courant dans un tableau, sans appel direct du navigateur à l'API K8s.
- [ ] 2. Couche de ressources générique (GVR)
  - Do: Découverte des API (`/api`, `/apis`) puis liste/get génériques par GroupVersionResource, avec une seule page « ressources » configurable.
  - Done when: tu listes pods, deployments, services et configmaps sans coder une page par type.
- [ ] 3. Multi-contextes et URL par cluster
  - Do: Lister les contextes du kubeconfig, ajouter `/c/:cluster` aux routes et transmettre le contexte choisi à chaque requête.
  - Done when: tu passes d'un contexte à l'autre et vois des ressources différentes.
- [ ] 4. Vue détail + YAML en lecture seule
  - Do: Page détail d'une ressource avec métadonnées, labels, conditions et YAML.
  - Done when: cliquer un pod affiche son YAML et ses métadonnées venant de l'API.
- [ ] 5. Temps réel avec `watch`
  - Do: Remplacer les listes ponctuelles par un watch Kubernetes, avec reconnexion et fallback polling.
  - Done when: `kubectl delete pod` met la liste à jour en quelques secondes sans rafraîchir.
- [ ] 6. Écriture : créer, éditer, supprimer
  - Do: Éditeur YAML, apply/delete, confirmations et affichage propre des erreurs 403 RBAC.
  - Done when: tu modifies un ConfigMap et supprimes un pod depuis l'UI.
- [ ] 7. Logs et exec
  - Do: Streaming des logs et terminal de pod via WebSocket + xterm.js.
  - Done when: tu ouvres les logs d'un pod et un shell dans un conteneur.
- [ ] 8. Premier déploiement « équipe »
  - Do: Image Docker, déploiement in-cluster (ServiceAccount + RBAC), auth (token ou OIDC) et isolation par utilisateur.
  - Done when: un coéquipier ouvre l'URL déployée, s'authentifie et ne voit que ce que son RBAC autorise.
