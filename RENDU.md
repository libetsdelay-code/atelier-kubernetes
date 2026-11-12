# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

> Environnement : l'atelier a été réalisé avec k3d v5.9.0 (k3s v1.35.5) et kubectl v1.37 sur une machine Linux locale avec Docker, à la place d'un GitHub Codespace. Les commandes sont identiques ; seule l'URL publique de la question 12 dépend du Codespace.

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > **3 nœuds**, tous `Ready` : `k3d-lab-server-0` (rôle `control-plane`, le « master » : API Server, scheduler, controller-manager, base de données du cluster) et `k3d-lab-agent-0` / `k3d-lab-agent-1` (rôle `<none>`, ce sont les workers qui exécutent les pods). Sur un cluster « vide », les pods de `kube-system` tournent déjà : `coredns` (DNS interne), `traefik` et `svclb-traefik-*` (Ingress Controller et son load balancer, un par nœud), `metrics-server`, `local-path-provisioner` (volumes), plus les jobs `helm-install-traefik*` en `Completed`.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : `k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1` (image `rancher/k3s`) et `k3d-lab-serverlb` (image `k3d-proxy`, qui publie `8081->80` et le port de l'API `6443`). Concrètement, **un nœud k3d est un conteneur Docker** qui fait tourner k3s (kubelet + containerd) : les pods sont des conteneurs lancés *à l'intérieur* de ces conteneurs. Le `serverlb` n'est pas un nœud Kubernetes, c'est le point d'entrée réseau du cluster.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** `web`, qui a créé un **ReplicaSet** `web-68d995574f`, qui a lui-même créé un **Pod** `web-68d995574f-xxxxx`. Le **Service** `web` (ClusterIP, port 80) vient de `kubectl expose`, pas de `create deployment`.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Le pod est passé en `Terminating`, et un **nouveau pod au nom différent** est apparu aussitôt (`web-68d995574f-r8p44` remplacé par `web-68d995574f-44ppn`) : `Pending`, `ContainerCreating`, puis `Running`. Le ReplicaSet veut 1 réplica et en constate 0 ; sa boucle de réconciliation recrée donc un pod pour revenir à l'état désiré. Le suffixe du ReplicaSet reste le même, seul le suffixe aléatoire du pod change.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > **Non.** Après la recréation, la page est redevenue *Welcome to nginx!*. Le système de fichiers d'un conteneur est éphémère : la modification n'existait que dans la couche inscriptible du conteneur supprimé, et le nouveau pod repart de l'image `nginx` d'origine.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > `kubectl port-forward svc/web` choisit **un seul pod** derrière le Service au démarrage et ouvre le tunnel vers lui. Quand ce pod meurt, le tunnel n'est pas redirigé vers le remplaçant ; `/tmp/web.log` affiche : `network namespace for sandbox ... is closed` puis `error: lost connection to pod`.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : il crée un objet et échoue (`AlreadyExists`) s'il existe déjà, sans pouvoir le mettre à jour. `kubectl apply -f` est **déclaratif** : il compare l'état décrit dans les fichiers à l'état du cluster, puis crée, modifie (`configured`) ou laisse en place (`unchanged`) chaque ressource. Il est rejouable à volonté et garde la dernière configuration appliquée dans l'annotation `kubectl.kubernetes.io/last-applied-configuration`. C'est ce qui permet de versionner l'état désiré dans Git.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` (`app: site`) désigne les pods, par leurs **labels**, qui reçoivent le trafic du Service ; Kubernetes en tient la liste à jour dans les **Endpoints** (`kubectl get endpoints site`). S'il ne correspond à aucun pod, le Service existe et garde son IP, mais il n'a **aucun endpoint** : les connexions échouent (refusées, ou 502/503 derrière l'Ingress), et `port-forward svc/...` refuse de démarrer faute de pod.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
NAME                    READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
site-5968b74899-4rzqw   1/1     Running   0          30s   10.42.2.7   k3d-lab-server-0   <none>           <none>
site-5968b74899-vlgd4   1/1     Running   0          40s   10.42.1.7   k3d-lab-agent-1    <none>           <none>
site-5968b74899-vrngg   1/1     Running   0          59s   10.42.0.9   k3d-lab-agent-0    <none>           <none>
   ```
   > Un pod par nœud : le scheduler les répartit. À noter : avec k3d, le nœud `server-0` (control plane) n'a pas de *taint* `NoSchedule` et reçoit donc aussi des pods applicatifs.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod était en **`ErrImagePull`** puis **`ImagePullBackOff`** (`0/1`). La cause exacte se trouve dans les Events de **`kubectl describe pod -l app=site`** : `Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found`. Ce tag n'existe pas sur Docker Hub. Corrigé avec `kubectl rollout undo deploy/site` ; `kubectl diff -f k8s/site/` ne renvoie ensuite plus aucun écart (image de nouveau `nginx:1.28`).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > **Oui** (HTTP 200). Le rolling update crée d'abord le nouveau pod et ne supprime un ancien pod que lorsque le nouveau est prêt (`maxUnavailable` 25 %, `maxSurge` 25 %). Comme le pod `nginx:9.99` n'est jamais devenu prêt, le déploiement s'est bloqué et les 3 anciens pods sont restés `Running`, dans les endpoints du Service.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://effective-train-v679g664q4j73r46-8585.app.github.dev/ (Mario déployé dans un GitHub Codespace avec k3d, port 8585 rendu public dans l'onglet PORTS)
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > `kubectl get deploy mario -o yaml` montre un Deployment standard, identique dans sa structure à ceux créés en CLI. Portainer y ajoute seulement des **labels** de suivi : `io.portainer.kubernetes.application.stack: mario`, `...application.owner: admin`, `...application.kind: content`, etc. Les deux outils parlent au **même composant : l'API Server (kube-apiserver)** du control plane. Portainer y accède depuis l'intérieur du cluster avec son ServiceAccount (lié en `cluster-admin`) ; `kubectl` y accède depuis l'extérieur avec le kubeconfig.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **A, Ingress** : `k8s/bonus/ingress.yaml` pointe vers le Service `site`, port 80 ; `curl localhost:8081` renvoie la page via Traefik. Supprimer **un** pod ne coupe plus rien, car l'Ingress répartit sur les endpoints, contrairement au port-forward. En revanche, en supprimant les **3 pods d'un coup**, j'ai obtenu `502 504` pendant ~2 s avant le retour à 200 : il faut au moins un pod prêt. Le tunnel, lui, n'a pas besoin d'être relancé.
    > **B, Probes et ressources** : `readinessProbe` HTTP sur `/:80` et requests/limits 50m/100m CPU, 32Mi/64Mi. Avec une probe sur `/nope`, le nouveau pod reste `Running` mais **`0/1`** (nginx répond 404, la probe échoue) : il n'entre jamais dans les endpoints, le rollout reste bloqué (`rollout status` en timeout), et les 3 anciens pods continuent de servir le site (HTTP 200). La probe empêche donc une version mal configurée de recevoir du trafic.
    > **C, Drain** : `kubectl drain k3d-lab-agent-0` marque le nœud `SchedulingDisabled` et évince ses pods (`site-...` et `local-path-provisioner`), aussitôt recréés sur `agent-1` / `server-0` ; les DaemonSets (`svclb-traefik`) restent en place. Après `uncordon`, **les pods ne reviennent pas** sur `agent-0` : Kubernetes ne rééquilibre pas les pods qui tournent déjà, et le scheduler ne place que les nouveaux pods. Le nœud ne resservira qu'au prochain pod créé (scaling, rollout, panne).
    > **D, Namespaces** : `kubectl create namespace prod` puis `kubectl apply -n prod -f k8s/site/` déploie une seconde copie indépendante (ConfigMap, Deployment et Service `site` portent les mêmes noms dans `default` et `prod` sans conflit), visible séparément dans Portainer en filtrant par namespace.
    > Remarque : l'API de Portainer 2.45 exige un `X-Setup-Token` (affiché dans les logs du pod) pour créer le compte admin.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
