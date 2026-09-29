# TP Jour 2 — Réponses aux questions de compréhension

## 1. Différence entre une image Docker et un conteneur Docker

Une **image** est un modèle figé, en lecture seule, construit à partir d'un `Dockerfile` : elle contient le système de fichiers (Python 3.12, les dépendances, `app.py`) et la commande de démarrage. Un **conteneur** est une instance *en cours d'exécution* d'une image : un processus isolé, avec sa propre couche inscriptible par-dessus l'image. On peut lancer plusieurs conteneurs à partir d'une même image (c'est ce que fait Kubernetes avec `mon-service:1.0` et ses 3 à 5 réplicas).

## 2. Pourquoi les données sont perdues si l'on supprime le volume, mais pas le conteneur

PostgreSQL écrit ses données dans `/var/lib/postgresql/data`, qui est monté sur le volume nommé `donnees_db`. Ce volume est géré par Docker **indépendamment du cycle de vie du conteneur** : `docker compose down` supprime les conteneurs mais garde le volume, donc au redémarrage le nouveau conteneur retrouve les données. `docker compose down -v` supprime aussi le volume : la base repart de zéro (et `init.sql` est rejoué, d'où le retour aux deux lignes initiales).

Vérifié pendant le TP : une ligne « Test persistance » ajoutée a survécu à `down`/`up`, puis a disparu après `down -v`.

## 3. Un Deployment avec `replicas: 3` perd un pod : que fait Kubernetes ? Pourquoi « déclaratif » ?

Le contrôleur du Deployment (via son ReplicaSet) constate que l'état réel (2 pods) ne correspond plus à l'état désiré (3 pods) et **crée immédiatement un nouveau pod** pour revenir à 3. C'est l'auto-réparation observée à l'étape 7.1.

On parle d'approche **déclarative** car on ne donne pas d'ordres (« lance un conteneur », « relance-le s'il tombe ») : on **décrit l'état souhaité** dans un fichier YAML (`replicas: 3`, image, ports…), et Kubernetes fait en permanence converger la réalité vers cet état (boucle de réconciliation).

## 4. Rôle du Service par rapport aux pods

Les pods sont éphémères : à chaque recréation ou scaling, ils reçoivent une nouvelle adresse IP. Le **Service** fournit un **point d'accès stable** (un nom DNS `api-service` et une IP virtuelle fixe) qui sélectionne dynamiquement les pods via le label `app: mon-service` et **répartit le trafic** entre eux. Les clients n'ont donc jamais besoin de connaître les IP des pods. Avec `type: NodePort`, il est en plus exposé sur un port du nœud, accessible depuis l'extérieur du cluster.

## 5. Deux limites de Docker seul (sans Kubernetes)

- **Pas d'auto-réparation à l'échelle d'un cluster** : si un conteneur ou la machine tombe, rien ne le recrée ailleurs automatiquement (au mieux une politique `restart` sur la même machine).
- **Pas de montée en charge ni de répartition de charge simples** : Docker/Compose tourne sur un seul hôte, on ne peut pas faire `scale --replicas=5` réparti sur plusieurs machines avec un load-balancer intégré.
- (Autres : pas de déploiements progressifs / rollback natifs, pas de gestion centralisée des secrets et de la configuration sur plusieurs machines.)
