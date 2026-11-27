# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > La commande affiche 3 noeuds. L'un est le master (server) les 2 autres sont les workers (agent).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : k3d-lab-server-0, k3d-lab-agent-0 et k3d-lab-agent-1 (image rancher/k3s), ainsi que k3d-lab-serverlb (image k3d-proxy), le load balancer qui relie le port 8081 au port 80 du cluster et donne accès à l'API Kubernetes (port 6443). Ce dernier n'est pas un nœud. Dans k3d, un nœud est concrètement un conteneur Docker qui exécute k3s (une distribution Kubernetes légère), et non une machine physique ou une VM dédiée.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > La commande crée un Deployment (deployment.apps/web). Celui-ci crée automatiquement un ReplicaSet (replicaset.apps/web-68d995574f), qui crée à son tour un Pod (web-68d995574f-nq8lh). Le service/web ne vient pas de cette commande mais de `kubectl expose`, et le service/kubernetes existait déjà dans le cluster.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > A la suite de la suppression du pod "web", le ReplicatSet en a recrée un immédiatement puisque qu'il est chargé de maintenir en vie N pods.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non, le message n'a pas survécu : après la suppression du pod, la page par défaut « Welcome to nginx! » est revenue. Le système de fichiers d'un conteneur est éphémère. Le Deployment (via son ReplicaSet) a créé un nouveau pod à partir de l'image nginx d'origine, qui ne contient pas la modification faite à la main dans l'ancien conteneur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Même lancé sur le Service (svc/web), le port-forward s'attache en réalité à un seul pod, choisi au démarrage du tunnel. Quand ce pod est supprimé, la connexion est perdue et le tunnel ne bascule pas automatiquement vers le nouveau pod. Il faut le relancer une fois le nouveau pod prêt.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` créée une ressource qui n'existe pas encore et renvoie une erreur si elle existe déjà. `kubectl apply` est déclaratif : on décrit l'état désiré dans un fichier YAML, et Kubernetes crée la ressource si elle n'existe pas ou la met à jour si le fichier a changé. 
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le selector d'un Service désigne, grâce aux labels (ici app: site), les pods qui doivent recevoir le trafic. Le Service leur fournit une adresse stable, même quand les pods sont remplacés. S'il ne correspond à aucun pod, le Service est quand même créé, mais il n'a aucun endpoint (visible avec `kubectl get endpoints site`) : les requêtes qui lui sont envoyées échouent, car aucun pod ne peut y répondre.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
   site-75775484b-gclm9   1/1     Running   0          85s     10.42.1.5   k3d-lab-agent-1    <none>           <none>
   site-75775484b-tpgr5   1/1     Running   0          4m59s   10.42.2.6   k3d-lab-server-0   <none>           <none>
   site-75775484b-vpmx9   1/1     Running   0          85s     10.42.0.8   k3d-lab-agent-0    <none>           <none>
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
   > Le nouveau pod était en erreur avec le statut ErrImagePull puis ImagePullBackOff : il ne pouvait pas démarrer. La commande `kubectl describe pod <nom-du-pod>` a donné la cause dans la section Events : l'image nginx:9.99 est introuvable car ce tag n'existe pas.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
   > Oui, le site restait accessible. Lors d'un rolling update, Kubernetes ne retire pas assez d'anciens pods pour mettre le service en danger tant que les nouveaux ne sont pas prêts. Ici, le nouveau pod n'est jamais devenu Ready : le déploiement est resté bloqué et les anciens pods ont continué à servir le site.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://opulent-dollop-ggqw4vp64wx2wwwv-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > On retrouve le Deployment avec `kubectl get deploy mario` (ou `kubectl get deploy mario -o yaml` pour le détail). Oui, Portainer et kubectl parlent au même composant : l'API Server (kube-apiserver), situé sur le control plane. kubectl l'appelle depuis le terminal, et Portainer, qui tourne dans le cluster, l'utilise depuis son interface web. Une ressource créée avec Portainer est donc aussi visible avec kubectl

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > Bonus A et B réalisés. L'Ingress (Traefik) expose le Service site sur le port 8081. Contrairement au port-forward, il n'est pas attaché à un seul pod : après la suppression d'un pod, le site reste accessible grâce aux autres réplicas. La readinessProbe permet de n'envoyer le trafic qu'aux pods prêts. Les requests réservent des ressources pour le placement des pods, et les limits plafonnent leur consommation de CPU et de mémoire.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
