# TP Kubernetes Day 2 & 3 — MySQL + phpMyAdmin

## Objectif

Ce projet consiste à déployer une stack **MySQL + phpMyAdmin** sur un cluster kind local, puis à la compléter avec des briques de production : `Secret`, `ConfigMap`, `PersistentVolumeClaim` et `NetworkPolicy`.

Le rendu demandé repose sur un dépôt Git contenant les manifestes Kubernetes, le fichier de création du cluster kind, un README explicatif, ainsi que des captures montrant le bon fonctionnement de l'application et des tests réseau.

## Périmètre réalisé

Les éléments suivants ont été mis en place :

- Un cluster kind nommé `esgi` avec 3 nœuds, dont un `control-plane` exposant le `NodePort 30080` sur `localhost:8080`.
- Un namespace dédié `database`, recommandé dans le sujet et valorisé dans la notation.[ :1]
- Un déploiement `MySQL` basé sur l'image `mysql:8`, exposé par un `Service` `ClusterIP` nommé `mysql`.
- Un déploiement `phpMyAdmin` basé sur l'image `phpmyadmin:latest`, avec 2 replicas et exposition par `Service` `NodePort`.
- Une externalisation de la configuration avec `Secret` pour le mot de passe MySQL et `ConfigMap` pour la configuration phpMyAdmin.
- Une persistance des données MySQL via un `PersistentVolumeClaim` monté dans `/var/lib/mysql`.
- Des `NetworkPolicies` limitant les flux réseau au strict nécessaire entre phpMyAdmin, MySQL et le DNS du cluster.

## Arborescence du dépôt

```text
.
├── kind-cluster.yaml
├── namespace.yaml
├── mysql-secret.yaml
├── mysql-pvc.yaml
├── mysql-deployment.yaml
├── mysql-service.yaml
├── phpmyadmin-configmap.yaml
├── phpmyadmin-deployment.yaml
├── phpmyadmin-service.yaml
├── networkpolicy-dns-egress.yaml
├── networkpolicy-pma-egress.yaml
├── networkpolicy-mysql-ingress.yaml
└── screenshots/
```

## Pré-requis

- macOS
- OrbStack avec Kubernetes/kind
- `kubectl`
- `kind`

Le sujet Jour 2 demande un cluster kind 3 nœuds opérationnel avant de commencer le projet MySQL + phpMyAdmin.

## Création du cluster

```bash
kind create cluster --config kind-cluster.yaml --name esgi
kubectl get nodes
```

Le fichier `kind-cluster.yaml` mappe le port `30080` du nœud control-plane vers `localhost:8080`, ce qui permet de joindre phpMyAdmin depuis le navigateur avec le `Service` `NodePort`.

## Déploiement

```bash
kubectl apply -f namespace.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f phpmyadmin-configmap.yaml
kubectl apply -f phpmyadmin-deployment.yaml
kubectl apply -f phpmyadmin-service.yaml
kubectl apply -f networkpolicy-dns-egress.yaml
kubectl apply -f networkpolicy-pma-egress.yaml
kubectl apply -f networkpolicy-mysql-ingress.yaml
```

Le README du Jour 3 précise que le déploiement doit être reproductible et que `kubectl apply -f .` est l'approche attendue pour un rendu propre et idempotent.

## Vérifications de base

### Vérifier les nœuds

```bash
kubectl get nodes
```

Résultat attendu : 3 nœuds en état `Ready`.

### Vérifier les ressources du namespace

```bash
kubectl -n database get pods
kubectl -n database get svc
kubectl -n database get pvc
kubectl -n database get networkpolicy
```

Le sujet Jour 3 demande explicitement de vérifier que le cluster tourne toujours avec au moins un Pod `mysql-*` et un Pod `phpmyadmin-*` en `Running` avant la finalisation.

## Accès à phpMyAdmin

URL d'accès attendue :

- `http://localhost:8080`

Paramètres de connexion :

- Serveur : `mysql`
- Utilisateur : `root`
- Mot de passe : valeur définie dans `mysql-secret.yaml`

Le sujet Jour 2 demande de tester l'accès à phpMyAdmin via `localhost:8080` et la connexion à MySQL avec le DNS interne `mysql`.

## Détail des manifestes

### 1. Namespace

Le namespace `database` permet d'isoler proprement les ressources du projet et correspond à la recommandation explicite du sujet Jour 3.

### 2. Secret MySQL

Le `Secret` permet de sortir le mot de passe root MySQL du Deployment afin d'éviter une valeur sensible en clair directement dans le manifeste applicatif.

### 3. ConfigMap phpMyAdmin

Le `ConfigMap` externalise la configuration non sensible de phpMyAdmin, notamment `PMA_ARBITRARY`, puis elle est injectée avec `envFrom` dans le Deployment.

### 4. PVC MySQL

Le `PersistentVolumeClaim` `mysql-data` demande `1Gi` en `ReadWriteOnce` et permet à MySQL de conserver ses données après suppression et recréation du Pod.

### 5. Services

- `mysql-service.yaml` expose MySQL en `ClusterIP` pour un usage interne au cluster.
- `phpmyadmin-service.yaml` expose phpMyAdmin en `NodePort` sur `30080` afin de le rendre accessible via le mapping kind vers `localhost:8080`.

### 6. NetworkPolicies

Les policies appliquées sont :

- `allow-dns-egress` : autorise les requêtes DNS vers `kube-system` sur le port 53 UDP/TCP.
- `allow-pma-to-mysql` : autorise la sortie de phpMyAdmin vers MySQL sur le port 3306.
- `allow-mysql-from-pma` : autorise l'entrée sur MySQL uniquement depuis phpMyAdmin sur le port 3306.

Le sujet recommande également un `deny-all` global dans le namespace avant de réautoriser les flux utiles ; cette logique a été étudiée durant la finalisation réseau.

## Test de persistance

La persistance a été validée selon la méthode demandée dans le sujet Jour 3 : création d'une base, suppression du Pod MySQL, puis vérification de la présence persistante des données après recréation du Pod.

Exemple de test :

```bash
kubectl -n database run mysql-client --rm -it --image=mysql:8 -- bash
mysql -h mysql -u root -p
```

Puis dans le client MySQL :

```sql
CREATE DATABASE tp_persistence;
USE tp_persistence;
CREATE TABLE t1 (id INT PRIMARY KEY);
INSERT INTO t1 VALUES (1);
```

Suppression du Pod MySQL :

```bash
kubectl -n database delete pod -l app=mysql
kubectl -n database get pods -w
```

Vérification après recréation :

```bash
kubectl -n database run mysql-client --rm -it --image=mysql:8 -- bash
mysql -h mysql -u root -p
SHOW DATABASES;
USE tp_persistence;
SELECT * FROM t1;
```

Résultat attendu : la base `tp_persistence` et la table `t1` sont toujours présentes, ce qui valide l'usage du PVC.

## Test des flux réseau

Le sujet Jour 3 demande de prouver que phpMyAdmin peut résoudre `mysql` et s'y connecter, tout en limitant les autres flux réseau via `NetworkPolicy`.

Exemple de test depuis phpMyAdmin :

```bash
kubectl -n database exec deploy/phpmyadmin -- sh -c "getent hosts mysql"
```

Résultat attendu : résolution DNS correcte du service `mysql.database.svc.cluster.local`.

## Captures fournies

Le sujet demande 3 captures dans le dossier `screenshots/`, en privilégiant les éléments réellement utiles au correcteur.

Captures recommandées :

- phpMyAdmin connecté à MySQL
- preuve de persistance après suppression/recréation du Pod MySQL
- tests réseau / NetworkPolicies

## Limites observées

Dans cet environnement macOS + OrbStack, l'accès navigateur via `localhost:8080` a pu devenir intermittent malgré un fonctionnement validé de phpMyAdmin à l'intérieur du cluster et une résolution correcte des services Kubernetes. Les captures et les vérifications internes montrent néanmoins que les Deployments, Services, PVC et échanges MySQL/phpMyAdmin fonctionnent comme attendu par le sujet.

## Commandes utiles

```bash
kubectl get nodes
kubectl -n database get pods,svc,pvc,networkpolicy
kubectl -n database logs deploy/mysql
kubectl -n database exec deploy/phpmyadmin -- sh -c "getent hosts mysql"
kubectl -n database delete pod -l app=mysql
```

## Conclusion

Le projet couvre les objectifs principaux des Jours 2 et 3 : déploiement applicatif sur Kubernetes, exposition réseau, externalisation de la configuration, persistance des données et sécurisation des flux réseau.

La base MySQL est accessible depuis phpMyAdmin via le Service interne `mysql`, les données persistent après redémarrage du Pod, et l'architecture repose sur des manifestes réutilisables prêts à être versionnés dans un dépôt Git.
