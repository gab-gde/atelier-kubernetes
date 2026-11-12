# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** GOUDE Gabin
**Groupe :** I2 APP RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds, tous `Ready` :
   > - `k3d-lab-server-0` avec le rôle `control-plane` : c'est le « master », il fait tourner l'API Server, le scheduler, le controller manager et la base d'état du cluster ;
   > - `k3d-lab-agent-0` et `k3d-lab-agent-1` (rôle `<none>`) : ce sont les workers, c'est là que le scheduler place les pods applicatifs.
   >
   > Version : k3s v1.35.5+k3s1, IP internes 172.18.0.2 à 172.18.0.4.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : `k3d-lab-server-0`, `k3d-lab-agent-0` et `k3d-lab-agent-1` (image `rancher/k3s`), plus `k3d-lab-serverlb` (image `k3d-proxy`), le load balancer qui publie le port 6443 de l'API et le port 8081 → 80.
   > Dans k3d, un nœud est donc simplement un **conteneur Docker** dans lequel tourne k3s. Il n'y a pas de vraie VM : les pods tournent avec containerd à l'intérieur de ces conteneurs.
   >
   > Sur le cluster « vide », `kubectl get pods -A` montre déjà des pods dans le namespace **`kube-system`** : `coredns` (DNS interne), `traefik` + `svclb-traefik` (Ingress Controller, un svclb par nœud), `metrics-server`, `local-path-provisioner` (volumes), et les jobs `helm-install-traefik` / `helm-install-traefik-crd` qui ont installé Traefik.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** (`deployment.apps/web`), qui a créé un **ReplicaSet** (`web-68d995574f`), qui a lui-même créé un **Pod** (`web-68d995574f-dq2vn`). Le **Service** `web` (ClusterIP 10.43.145.209, port 80), lui, vient de la commande `kubectl expose`, pas de `create deployment`.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Dans le `kubectl get pods -w`, l'ancien pod `web-68d995574f-dq2vn` est passé en `Terminating` et presque en même temps un nouveau pod `web-68d995574f-kgznq` est apparu (`Pending` → `ContainerCreating` → `Running` en environ 8 s). Le nom a changé, seul le préfixe du ReplicaSet est resté le même.
   > C'est le ReplicaSet qui fait ça : il doit maintenir 1 réplica, il voit qu'il en manque un et il en recrée un (boucle de réconciliation entre état désiré et état réel).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non. Après la suppression et la relance du tunnel, on retombe sur la page « Welcome to nginx! ». Le nouveau pod est créé à partir de l'image `nginx` d'origine et le système de fichiers d'un conteneur est éphémère : ce que j'avais écrit avec `kubectl exec` n'existait que dans l'ancien conteneur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Parce que `kubectl port-forward svc/web` ne passe pas vraiment par le Service : il choisit **un pod** derrière le Service au lancement et ouvre le tunnel directement vers lui. Quand ce pod est supprimé, le tunnel n'a plus de cible. Dans `/tmp/web.log` on voit `error forwarding port 80 to pod ...` puis `error: lost connection to pod` (et `curl` renvoie une réponse vide).

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : on demande de créer un objet, et ça échoue s'il existe déjà. `kubectl apply -f` est **déclaratif** : on donne l'état voulu dans un fichier, Kubernetes compare avec ce qui existe et crée ou modifie seulement ce qui change (on l'a vu avec `configmap/site-html configured` et `service/site unchanged`). On peut le relancer autant de fois qu'on veut, et les fichiers peuvent être versionnés dans Git.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` (`app: site`) dit au Service quels pods doivent recevoir le trafic : tous les pods qui ont ce label deviennent ses endpoints (`kubectl get endpoints site` → `10.42.0.6:80`). S'il ne correspond à aucun pod, le Service existe quand même mais il n'a aucun endpoint, donc les requêtes n'arrivent nulle part (pas d'erreur à la création, c'est ce qui rend le problème difficile à voir).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
   site-848dbc487f-7q2wf   1/1     Running   0          25s   10.42.1.6   k3d-lab-agent-1    <none>           <none>
   site-848dbc487f-7qtg6   1/1     Running   0          7s    10.42.0.7   k3d-lab-agent-0    <none>           <none>
   site-848dbc487f-nbs7z   1/1     Running   0          7s    10.42.2.7   k3d-lab-server-0   <none>           <none>
   ```
   > Un pod par nœud. Le scheduler a aussi utilisé `k3d-lab-server-0` : avec k3s, le control plane n'a pas de taint `NoSchedule` par défaut, donc il peut recevoir des pods comme un worker.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod (`site-7ddc5d9684-qs5zx`) était en **`ImagePullBackOff`** (en alternance avec `ErrImagePull`), `0/1`.
    > La cause exacte est dans les Events de `kubectl describe pod -l app=site` : `Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found`. Le tag n'existe pas sur Docker Hub. `kubectl logs` ne donne rien, puisque le conteneur n'a jamais démarré.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui, le site répondait toujours (HTTP 200 avec ma page). Pendant un rolling update, Kubernetes crée d'abord un nouveau pod et ne supprime un ancien que quand le nouveau est prêt. Le nouveau pod n'étant jamais prêt, les 3 anciens pods en `nginx:1.28` sont restés `Running` (`kubectl get deploy site` → `3/3`, `UP-TO-DATE 1`, `AVAILABLE 3`) et le Service continuait à envoyer le trafic vers eux.
    > Après `kubectl rollout undo deploy/site`, on est revenu en `nginx:1.28` et `kubectl diff -f k8s/site/` ne montrait plus aucune différence avec les fichiers.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://literate-guacamole-97rwjr9995762pq79-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui, `kubectl get deploy mario -o yaml` le montre bien, c'est un Deployment normal. La seule différence avec ceux créés en ligne de commande, ce sont les labels ajoutés par Portainer : `io.portainer.kubernetes.application.stack: mario`, `io.portainer.kubernetes.application.owner: admin`, `...stackid: "1"`, etc. (alors que `site` n'a que `app: site`).
    > Portainer et `kubectl` parlent tous les deux à l'**API Server** (kube-apiserver) du control plane. Portainer tourne dans le cluster avec le ServiceAccount `portainer-sa-clusteradmin` et appelle la même API REST que `kubectl` : c'est pour ça que les nœuds, le Deployment `site`, ses 3 pods et la ConfigMap `site-html` apparaissaient directement dans Portainer.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **A — Ingress** : Ingress `site` → Service `site` port 80, classe `traefik`. Le site répond sur le port 8081 sans aucun port-forward. Si je supprime un pod, toutes les requêtes restent en 200 (le trafic passe par le Service et donc par les pods restants). En supprimant les 3 pods d'un coup, j'ai eu 2 erreurs (502 puis 504) le temps que les nouveaux pods démarrent, puis ça revient tout seul.
    >
    > **B — Probes et ressources** : `readinessProbe` HTTP sur `/` port 80, requests 50m CPU / 32Mi, limits 100m / 64Mi. En testant avec `/nope`, le nouveau pod reste `0/1 Running` avec `Readiness probe failed: HTTP probe failed with statuscode: 404`, il n'est jamais ajouté aux endpoints du Service et le rolling update se bloque. Les anciens pods continuent donc à servir le site. Je suis ensuite revenu au bon manifeste.
    >
    > **C — Drain** : après `kubectl drain k3d-lab-agent-0`, le nœud passe en `Ready,SchedulingDisabled` et ses pods sont évincés (le pod `site` mais aussi traefik et local-path-provisioner). Le pod `site` a été recréé sur `k3d-lab-agent-1`. Après `kubectl uncordon`, les pods **ne reviennent pas** sur agent-0 : Kubernetes ne rééquilibre pas les pods qui tournent déjà, le scheduler n'intervient qu'à la création d'un pod. Le nœud ne resservira qu'aux prochains pods créés (scaling, rolling update, suppression...).
    >
    > **D — Namespaces** : namespace `prod` créé, et `kubectl apply -n prod -f k8s/site/` déploie une deuxième copie complète (ConfigMap, Deployment 3/3, Service) avec les mêmes noms qu'en `default` sans conflit. Dans Portainer, `prod` apparaît comme un namespace séparé avec ses propres applications.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
