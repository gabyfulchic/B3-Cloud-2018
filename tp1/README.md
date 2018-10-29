# TP1 Cloud : Architectures complexes cloud-like

Le but de ce TP est de déployer une stack applicative complexe reflétant les enjeux modernes du cloud :
* provisionning automatisé
  * répétabilité du processus

* déploiement applicatif (+ packaging et distribution)
  * simple
  * rapide
  * scalable

* accessibilité
  * on accède aux services avec performance
  * technos dédiées

* automatisation
  * répétabilité
  * conformité
  * sécurité 

* robustesse
  * stockage redondé
  * applications hautement disponibles

* récolte et visualisation de data (+ alerting?)

**!!! PRENEZ DES NOTES DE VOS AVANCEES !!!**  

Le rendu s'effectuera à travers un dépôt GitHub (ou autres, mais un dépôt `git` que je peux atteindre). Le format `md` est donc tout indiqué pour votre compte-rendu, mais vous restez libres :)

# Notions/Technos
* Système
  * CoreOS CoreOS (OS)

* Provisionning/Templating
  * CoreOS Ignition (cloud config bootstrap)
  * Vagrant (VM provisionning)
    * [Terraform](https://www.terraform.io/) would have been better but does not pair very well with dev hypervisor like VBox

* Applicative packaging/Deployment
  * Docker (packaging)
  * Docker Swarm (deployment)
  * Docker Registry (image distribution)

* Storage
  * CEPH (distributed file-system)

* Monitoring/Métrologie
  * Weave Cloud (monitoring SaaS)
  * Prometheus (local data storage and distribution)
  * Grafana (data visualisation)
  * + autres

* Exposition
  * Traefik (microservices-oriented reverse proxy)
  * Keepalived (Virtual IP)

* Un peu sous le capot, entre autres, on s'attardera pas dessus : 
  * systemd
  * torcx
  * CoreOS Ignition

# Prerequisites

* Récupérez le projet `git` https://github.com/coreos/coreos-vagrant.

* explorez un peu le `Vagrantfile`

* customisez le Vagrantfile pour : 
  * ajouter un dique de 10Go
* pour gagner du temps par la suite, vous pouvez aussi customisez la box utilisée afin d'y intégrer les images `docker` suivantes :
  * [`ceph/daemon`](https://hub.docker.com/r/ceph/daemon/)
  * [`osixia/keepalived:1.3.5`](https://github.com/osixia/docker-keepalived)
  * [`traefik:latest`](https://hub.docker.com/_/traefik/)

* Créez ensuite **5 machines** à l'aide de ce `Vagrantfile` (ça se fait facilement à l'aide d'une variable déjà créée dans le fichier `Vagrantfile`).

# Docker Swarm

[Docker est déjà installé par défaut sur CoreOS](https://coreos.com/os/docs/latest/getting-started-with-docker.html). Inspectez sa configuration (unité de service `systemd` et `docker info`).  

Pour rappel, [Docker Swarm](https://docs.docker.com/engine/swarm/#feature-highlights) permet de mettre en place de la haute disponibilité au niveau du lancement et de l'entretien des conteneurs. Nous allons mettre en place **un swarm de 5 noeuds** : 3 managers et 2 workers. Le cluster restera sain tant que 2 managers seront en vie.

## Mise en place

Configurez :
* le `experimental` mode de Docker pour pouvoir récupérer des métriques (voir [dockerd config](https://docs.docker.com/engine/reference/commandline/dockerd/))
  * ajoutez aussi la clause `metrics_addr` et sa valeur `0.0.0.0:9323`

* **un swarm avec 3 managers et 2 workers.**
```
# Sur votre premier manager :
docker swarm init --advertise-addr <IP_HOST_ONLY>

# Génère une commande à taper sur les autres managers
docker swarm join-token manager

# Génère une commande à taper sur les workers
docker swarm join-token worker
```

## Commandes de base

* `docker node ls`

* `docker stack ls`
* `docker stack deploy MA_STACK -c docker-compose.yml`
  * déploie une stack depuis un fichier compose
* `docker stack services MA_STACK`
  * liste les services d'une stack
* `docker stack ps MA_STACK`
  * liste les conteneurs d'une stack et leur état

* `docker service ls`
* `docker service logs SERVICE`
  * affiche les logs d'un service (basé sur `docker logs`)

# Dumb Service - Part 1

Tout au long du TP, nous déploierons ce service, mais en changeant notre process de déploiement à chaque fois.  

Tous nos services, nos applicatifs, tourneront sous la forme de stacks ou services Swarm. On crée un `docker-compose.yml` pour chacune des stacks. On trouvera aussi un répertoire de donnée pour chacune d'entre elles.

* utilisez le `compose.yml` fourni afin d'avoir une stack de test (provient de la correction du TP2 Conteneurisation). Il pop l'archi suivante : `BDD <--- AppPython <--- NGINX`
* les images utilisées dans le `docker-compose.yml` doivent être accessibles sur tous les noeuds.

```
docker stack deploy python_dirty_app -c docker-compose.yml
```

* une fois démarrée l'appli devrait être joignable sur le port 8080 de votre cluster

# Weave Cloud

[Weave Cloud](https://cloud.weave.works/signup) permet d'accéder à des fonctionnalités de monitoring et métrologie directement depuis une interface dans le cloud (du SaaS donc :)). 

* utiliser [Weave Cloud](https://www.weave.works/docs/cloud/latest/install/docker-swarm/) pour monitorer votre déploiement Swarm
  * il vous faudra un compte Weave
  * inscrivez-vous (n'hésitez pas à utiliser un mail jetable :) )
  * demandez une connexion à une instance de Docker Swarm pour récupérer votre token
  * utilisez un conteneur Docker Weave([toujours la même page](https://www.weave.works/docs/cloud/latest/install/docker-swarm/)) avec votre token
    * celui-ci va déployer  une stack sur votre swarm et s'auto-détruire
  * le conteneur Weave va utiliser votre swarm pour lancer des services.
  * **Q1 : comment ce conteneur a-t-il pu lancer des conteneurs sur votre hôte ?**  

Une fois lancé, RDV sur l'interface graphique de Weave pour voir la magie. Explorez un peu y'a une tonne d'infos. 

# A. CEPH ooooouuuuu...

## Présentation

[CEPH](https://ceph.com/) est un outil permettant de mettre en place (notamment) un système de fichiers distribué ; plutôt que d'avoir une partition locale utilisé sur un filesystem local, nous allons mettre en commun des partitions (en réalité, des blocs) à travers le réseau, et rendre le tout accessible sur tous nos hôtes comme une simple partition.  

**CEPH est un outil complexe**, il se décompose en plusieurs entités que nous installerons séparément : 
* *monitors* : sont en charge de maintenir les cartes représentant le cluster (crucial pour que les démons CEPH soit synchro)
  * 3 suffiront pour notre petit lab
* *managers* : récupère et expose des métriques, ainsi que l'état du cluster
  * 3 managers aussi, sur les mêmes noeuds que les monitors
* *OSD* : le coeur de l'outil, les OSDs sont en charge de stocker de la donnée (et d'autres opération avancées comme la réplication, la restauration etc.)
  * sur TOUS les noeuds où du stockage est dispo, sur tous nos noeuds donc
* *MDS* : ce sont les serveurs de metadata, indispensable pour utiliser de façon simple le filesystem de type `ceph`
  * sur tous les noeuds

Le fonctionnement en détail de CEPH dépasse le cadre du cours, nous allons donc mettre en place un setup simple.  

Vuuuu qu'on est des guerriers **on va le mettre en place à l'aide de Docker** :)

## Mise en place

**Suivez bien les étapes, c'est crucial.**

Une fois lancés, vous aurez accès à la commande `ceph` à l'intérieur des conteneurs. Vous pouvez vous en servir pour vérifier l'état du cluster au long de l'install, par exemple : 
* `ceph status` affiche l'état du cluster
* `ceph df` affiche l'état d'occupation des pools de stockage CEPH

### 1. Monitors

* Sur votre premier host

```
docker run -d --net=host \
--restart always \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=<IP_HOST_ONLY>  \
-e CEPH_PUBLIC_NETWORK=<HOST_ONLY_CIDR_NETWORK> \
--name="ceph-mon" \
ceph/daemon mon
```
* `MON_IP` c'est pour "*monitor* IP"
* suite à ça, déplacer tout le contenu de `/etc/ceph/` sur **tous les autres noeuds**
* exécuter de nouveau la commande `docker run` ci-dessus sur deux autres noeuds (ce sera vos 3 *monitors*)
* n'oubliez pas de changer la variable `MON_IP` sur chacun de vos 3 *monitors*
* check :
```
docker exec ceph-mon ceph status
  cluster:
    id:     faba0138-849f-491d-8ba3-3ce1b92cff19
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs: 
```

### 2. Managers

Sur les 3 mêmes hôtes que les *monitors* :

```
docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
--name="ceph-mgr" \
--restart=always \
ceph/daemon mgr
```
* check
```
docker exec ceph-mon ceph status
  cluster:
    id:     faba0138-849f-491d-8ba3-3ce1b92cff19
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: core-01(active), standbys: core-03, core-02
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
```

### 3. OSDs

Rentrez dans un conteneur *monitor* en utilisant un `bash` (commande `docker exec`).  

Récupérez l'output de la commande `ceph auth get client.bootstrap-osd` et stockez le dans un fichier.  

Sur tous les noeuds : 
* copier le contenu du fichier précédemment créé dans `/var/lib/ceph/bootstrap-osd/ceph.keyring`
  * effacez s'il existe déjà, créez s'il n'existe pas
* puis :
```
docker run -d --net=host \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /dev/:/dev/ \
-e OSD_FORCE_ZAP=1 \
-e OSD_DEVICE=<CHEMIN_VERS_NEW_DISK_IN_/dev> \
-e OSD_TYPE=disk \
--name="ceph-osd" \
--restart=always \
ceph/daemon osd_ceph_disk
```
* n'oubliez pas de changer `CHEMIN_VERS_NEW_DISK_IN_/dev` par `/dev/sdb` par exemple


* check cluster status
```
docker exec ceph-mon ceph status
  cluster:
    id:     9f1ed298-688b-4143-bc53-5b869519ba73
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum core-01,core-02,core-03
    mgr: core-02(active), standbys: core-03, core-01
    osd: 5 osds: 5 up, 5 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   10 GiB used, 37 GiB / 47 GiB avail
    pgs:
```

### 4. MDS

* Sur tous les noeuds : 
```
docker run -d --net=host --name ceph-mds --restart always -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph -e CEPHFS_CREATE=1 -e CEPHFS_DATA_POOL_PG=128 -e CEPHFS_METADATA_POOL_PG=256 ceph/daemon mds
```

* Sur un noeud, pour éviter de péter vos machines :

```
ceph osd pool set cephfs_data size 2
ceph osd pool set cephfs_metadata size 2
ceph osd set nodeep-scrub
```

### 5. Config finale

* Dans un des conteneurs *monitor*, on va créer un secret qui nous permettra d'accéder au volume CEPH sur tous nos noeuds :
```
ceph auth get-or-create client.dockerswarm osd 'allow rw' mon 'allow r' mds 'allow' > keyring.dockerswarm
cat keyring.dockerswarm
```

* Puis sur tous les noeuds :
```
mkdir /data
echo "<MONITOR_IPS_COMMA_SEPARATED>:6789:/      /data/      ceph      name=dockerswarm,secret=<YOUR_SECRET_HERE>,noatime,_netdev 0 2" > /etc/fstab
mount -a`
```

**Le répertoire `/data` devrait être accessible sur les noeuds.**  

**Conseil : utilisez le pour stocker TOUTES vos configurations par la suite, y compris vos confs CEPH**

### 6. Un peu de réflexion

* **Q2 : expliquer rapidement le principe d'un filesystem distribué**  

* **Q3 : proposer une façon d'automatiser cette conf CEPH** (Vagrant ? Swarm stack ? autres ?)

* nous allons utiliser ce répertoire pour stocker des données sur le FS des noeuds Docker. Serait-il possible que nos conteneurs utilisent directement les volumes CEPH, sans passer par un volume de l'hôte ? Illustration
  * actuel : CEPH --*MDS*--> Host --`run -v`--> conteneur
  * demandé : CEPH --*MDS*--`run -v`--> conteneur

# B..... NFS
Ou un simple partage NFS. Je ne donnerai pas d'instructions pour cette partie (très simple normalement). Pour les non-initiés, NFS (sobrement Network FileSystem) et un... système de fichiers sur le réseau :|. Il existe plusieurs articles sur internet qui expliquent comment le mettre en place, c'est plus easy qu'un CEPH.  

Attention en revanche : NFS peut mal supporter les accès concurrents, surtout en écriture (vous pouvez monter votre partition NFS uniquement en lecture).

# Registry - Part 1

Pour lancer un service à travers le swarm, tous les noeuds doivent pouvoir pull le conteneur. Plusieurs choix alors :
* se log sur tous les noeuds et récupérer l'image souhaitée, partout (`docker pull`, ou archive, ou autres)
* publier l'image sur un registre public/joignable sur internet
* **monter un registre soi-même. On part sur cette option.**  

Le [Docker Registry](https://docs.docker.com/registry/) (ou simplement *registre*) doit être joignable sur tous les noeuds du Swarm. 

Pour rappel le Docker Registry permet d'héberger et distribuer des images Docker.  

On va en lancer un sur notre swarm, qui pourra être joignable en local (`127.0.0.1:5000`) sur tous les hôtes : 

```
docker service create --name registry --publish published=5000,target=5000 registry:2
```
Normalement, un `curl 127.0.0.1/v2/`  devrait fonctionner et retourner `{}` sur tous les hôtes.  

Expliquez :
* **Q4 : où est lancé le service réellement ? (sur quel hôte, et comment on fait pour savoir ?)** Combien y'a-t-il de conteneurs lancés ? 
* **Q5 : pourquoi le service est accessible depuis tous les hôtes ?** Documentez vous sur internet.

# Dumb Service - Part 2

Faites tourner le Dumb Service mais :
* Hébergez les images du Dumb Service dans le registre
* Utilisez le répertoire `/data` pour stocker ses données (`docker-compose.yml`, configurations, applications, données)
* Observer son évolution sur Weave Cloud

```
cd /data/python_app/
docker-compose build
docker-compose push
docker stack deploy python_dirty_app -c docker-compose.yml
```

# Keepalived

Actuellement, un service est joignable sur toutes les IPs des membres du Swarm. Ok, mais on dit quoi à notre DNS ?  

On va mettre en place une **IP Virtuelle**, que porteront tous nos serveurs, à l'aide de [Keepalived](http://www.keepalived.org/). Vu qu'on est des hipsters : conteneurs !

En admettant la config suivante : 
* IP Virtuelle voulue sur `172.17.8.100`
* IPs des 5 hôtes : `172.17.8.101`, `172.17.8.102`, `172.17.8.103`, `172.17.8.104`, `172.17.8.105`
* on obtient la commande suivante pour lancer Keepalived sur un noeud :

```
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --net=host \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.17.8.101', '172.17.8.102', '172.17.8.103', '172.17.8.104', '172.17.8.105']" \
  -e KEEPALIVED_VIRTUAL_IPS=172.17.8.100 \
  -e KEEPALIVED_PRIORITY=200 \
  osixia/keepalived:1.3.5
```

* Lancez la commande sur tous vos hôtes, en modifiant la priorité à chaque fois
  * **une fois fait, vous n'accéderez à votre cluster plus qu'avec l'ip virtuelle**

* Expliquez :
  * **Q6 : le principe de priorité au sein de Keepalived**
  * **Q7 : le fonctionnement simplifié de `vrrp`** (schéma si vous voulez)

# Show me your metrics

Je veux un tableau de bord. Avec des chiffres. **Partout.**  

Ici, afin d'avoir quelque chose de rapidement fonctionnel, on va réutiliser un projet existant et très bien configuré : [swarmprom](https://github.com/stefanprodan/swarmprom).  

Copy/paste du `README.md` du projet :

* prometheus (metrics database) http://<swarm-ip>:9090
* grafana (visualize metrics) http://<swarm-ip>:3000
* node-exporter (host metrics collector)
* cadvisor (containers metrics collector)
* dockerd-exporter (Docker daemon metrics collector, requires Docker experimental metrics-addr to be enabled)
* alertmanager (alerts dispatcher) http://<swarm-ip>:9093
* unsee (alert manager dashboard) http://<swarm-ip>:9094
* caddy (reverse proxy and basic auth provider for prometheus, alertmanager and unsee)

* Je vous laisse suivre la doc du `README.md` pour un déploiement simple. Baladez-vous un peu, surtout sur Grafana qui offre de précieuses visualisations.  

* Expliquez 
  * **Q8 : ce qu'est un 'collector' dans ce contexte**
  * **Q9 : un peu plus en détail le fonctionnement de [Prometheus](https://prometheus.io/)**

# Traefik

[Traefik](https://traefik.io/) est un reverse proxy dédié aux environnements micro-service.   

Au sein d'un Swarm, il est capable de détecter dynamiquement les nouveaux services qu'il doit servir, grâce à un système de *labels*.  

Mettons le en place : 
* architecture de fichiers/dossiers
```
mkdir /data/traefik
mkdir /data/traefik/certs
touch /data/traefik/traefik.toml
touch /data/traefik/docker-compose.yml
```
* générer une paire de clé/cert auto-signé. Utilisez une wildcard pour votre *COMMON NAME* (exemple : `*.b3.swarm`) : 
```
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /data/traefik/certs/b3.swarm.key -out /data/traefik/certs/b3.swarm.crt
```
* editer le fichier [`/data/traefik/traefik.toml`](./traefik/traefik.toml)
* editer le fichier [`/data/traefik/docker-compose.yml`](./traefik/docker-compose.yml)

* lancer une stack Traefik à l'aide du `docker-compose.yml`

* explorer l'interface de Traefik
* **Q10: expliquez un peu son fonctionnement**

# Dumb Service - Part 3

Adaptez le `docker-compose.yml` de l'app Python pour tourner derrière Traefik en HTTPS uniquement.

# Registry - Part 2

Faites tourner le registre en HTTPS derrière Traefik.

# Registry - Part 3

Faites tourner une stack [Harbor](https://goharbor.io/) plutôt qu'un Registry seul.

# Backup

* **Q11 : Proposez une solution permettant de sauvegarder nos applications**. On veut garder :
  * les configurations
  * les données
  * les outils de déploiement
  * les applications elles-mêmes

* Ici j'attends un peu de réflexion sur la question. Nous avons déjà des outils pour lancer n'importe quel applicatif (stack CoreOS + Docker + Swarm), et un répertoire de données accessibles sur tous nos hôtes (CEPH, NFS). Je ne veux pas un script `bash` qui fait du `rsync` :)

# Récap

* mettez-moi tout ça au clair, proposez : 
  * une mini-doc permettant de déployer un nouveau service sur le Swarm, qui sera monitoré, sauvegardé, servi en HTTPS, et en HA
  * une mini-doc permettant d'ajouter un noeud au swarm (rappel : il doit aussi avoir accès aux répertoire de données, être sauvegardé et monitoré)
