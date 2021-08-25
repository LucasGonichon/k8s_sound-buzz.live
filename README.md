# PROJET SOUND-BUZZ : Déploiement d'un application de streaming multimedia à l'aide de micro-services.

> Qu'est ce que Kubernetes (k8s) ? Qu'est ce qu'un cluster ? Une node ? Un pod ? Les réponses ici : [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/).

## Création des VM Hyper-V
On commence par créer des machines virtuelles pour composer notre futur cluster k8s.

> On utilise un système d'exploitation Linux pour héberger les différents composants de notre **cluster** : [Ubuntu Server 20.04 LTS](https://mirrors.ircam.fr/pub/ubuntu/releases/20.04.2/ubuntu-20.04.2-live-server-amd64.iso).

Dans notre environement, les machines virtuelles sont hébergées avec hyper-V. Nous attribuons 2048Mo de RAM, 20Go d'espace disque et 2 CPU virtuels pour chaque VM. On ajoutera à chaque VM une carte réseau utilisant le même réseau que le reste de l'infrastructure : ```172.16.0.0/24```.

Nous utilisons des systèmes invités Linux avec Hyper-V, pour réussir à installer l'OS correctement, il faut bien désactiver le "démarrage sécurisé" dans les options de chaque VM : ![](img/secure_boot.PNG)

Nous créons 4 VM :
- k8s-control
- k8s-node1
- k8s-node2
- k8s-node3

> Il est possible d'ajouter autant de nodes que l'on souhaite au cluster : on peut répéter la même installation en adaptant le nom de la machine et son addresse ip.

### Installation d'ubuntu server
Dans cette rubrique, je ne prendrais pour exemple que la VM "k8s-control", il faudra cependant effectuer les opérations sur chaque VM, en adaptant certains paramètres le cas échéant (noms d'hôtes, addresses ip, ...).

Au premier lancement, il nous est proposé plusieurs options, nous choisissons *Install Ubuntu Server*.

> Nous ferons l'installation en Anglais, mais vous pouvez la faire dans une autre langue, cela n'aura pas d'impact sur la suite, modulo la traduction des menus. Si aucune instruction n'est fournie ici concernant un menu, laissez les options par défaut.
>
> Vous pouvez choisir la configuration de votre clavier dans l'écran suivant le choix de la langue.

Le premier menu sur lequel nous effectuons des opération est le menu portant sur la configuration de l'interface réseau de la machine. On renseigne notre **passerelle par défaut** *(Gateway)*, notre serveur **dns** local *(Name servers)*, ainsi que notre **domaine** *(Search domains)* selon les caractéristiques de notre infrastructure, et l'**addresse ip** *(Address)* de la machine selon la plage que l'on a choisi pour notre **cluster**.

![](img/dhp4_off.PNG)
![](img/ipv4_config.PNG)

> Attention à bien donner une addresse ip différente pour chaque VM, on prendra les ip à partir de .150 ici, mais libre à vous de choisir votre propre plage pour votre **cluster**.

Le reste des options sera laissé par défaut, à l'exception de 2 menus :
- On active le serveur open-SSH (facultatif, permet par la suite de se connecter en ssh au serveur plutôt que de passer par l'interface Hyper-V).
- Le menu "Profile setup", dont la configuration sera comme suit (attention à bien adapter le nom de la machine) : ![](img/profile_setup.PNG)
  *Vous êtes libre d'utiliser un nom d'utilisateur et un mot de passe de votre choix. Ici "ubuntu" n'est qu'un exemple. Ce compte correspond au compte administrateur par défaut de la machine.*

Une fois l'installation completée et les mises à jour de sécurité installées, il ne vous reste qu'à redémarrer chaque machine (pensez à éjecter le média d'installation).

### Configuration initiale
> Si vous avez choisi d'activer le serveur open-SSH, vous pouvez ajouter votre clé publique au fichier ```~/.ssh/authorized_keys``` pour vous connecter en ssh sans entrer votre mot de passe à chaque fois.
>
> Pour savoir comment générer une paire de clés, on sort du cadre de cette procédure d'installation. Pour plus d'informations sur les clés ssh : [SSH keys](https://www.ssh.com/academy/ssh/key)

Avant d'installer k8s, il nous faut un environement de containerisation *(container runtime)*. On va installer [docker](https://www.docker.com/) pour notre **cluster** (une alternative serait par exemple [containerd](https://containerd.io/), aujourd'hui l'option par défaut de [kubernetes](https://kubernetes.io/)).
```shell
curl -sSL get.docker.com | sh
sudo usermod -aG docker ubuntu
```
> La 2e commande sert à éviter de devoir utiliser notre mot de passe à chaque interraction avec le moteur **docker**. Pensez bien à remplacer *ubuntu* par le nom d'utilisateur que vous avez choisi lors de l'installation de la VM.

On configure des options du docker daemon dans un fichier daemon.json (on le créé, il n'est pas présent par défaut) ```/etc/docker/daemon.json``` :
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```
> Plus d'infos sur ce fichier et ses options : [Dockerd](https://docs.docker.com/engine/reference/commandline/dockerd/).
>
> Pourquoi **systemd** et pas **cgroupfs** ? Choix arbitraire, **cgroupfs** est le choix par défaut sur docker.
>
> Pourquoi **overlay2** ? https://docs.docker.com/storage/storagedriver/select-storage-driver/

On autorise le "port forwarding" (nécessaire pour générer l'environement réseau des containers) : on modifie pour cela le fichier ```/etc/sysctl.conf``` en modifiant la ligne ```#net.ipv4.ip_forward=1``` en : ```net.ipv4.ip_forward=1``` (on dé-commente l'option en retirant le #). Cela sera nécessaire pour que les **containers** et les **pods** puissent communiquer entre eux et avec l'extérieur.

On peut maintenant redémarrer le serveur et tester si docker est correctement installé avec la commande : ```docker run hello-world```.
Vous devriez avoir ce message à l'écran si tout est correctement installé :
```shell
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Vous pouvez aussi vérifier que le service docker est bien actif avec ```systemctl status docker```. Par défaut il devrait être **active** et **enabled**.

### Installation de kubernetes
On peut maintenant installer kubernetes sur chaque VM. On commence par ajouter le **repository** kubernetes en créant le fichier ```/etc/apt/sources.list.d/kubernetes.list``` avec le contenu suivant :
```
deb http://apt.kubernetes.io/ kubernetes-xenial main
```
On récupère la clé GPG pour valider le **repository** :
```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
> **kubernetes** est à la base un projet dévelloppé par *google*, on récupère donc les sources depuis leurs serveurs.

On peut désormais mettre à jour notre index puis installer **k8s** :
```shell
sudo apt update
sudo apt install kubeadm kubectl kubelet
```

## Création du cluster
Avant de commencer, on désactive le *swap* sur chaque VM :
```shell
sudo swapoff -a
```

### Controler
Uniquement sur la VM *k8s-controler*, on execute la commande suivante :
```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> Le réseau choisi est le réseau interne au cluster, il doit être différent de celui de votre infrastructure.

La commande devrait sortir dans le terminal une ligne de ce type :
```kubeadm join 172.16.0.150:6443 --token 61z7u0.owjzoqjtzvm3djov --discovery-token-ca-cert-hash xxx```.

Gardez cette commande de côté, elle servira plus tard.
> Attention : Ce sont des données assez sensibles d'un point de vue sécurité, faites attention à ne pas partager cette commande, et à ne pas la conserver une fois qu'elle n'est plus utile.

Lancez maintenant les commandes suivantes :
```shell
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> Elles se trouvent aussi dans la sortie de la commande précédente, si elles sont différentes chez vous, adaptez cette étape avec les commandes sorties à l'étape précédente.

On doit maintenant installer un driver pour gérer le réseau au sein du cluster. On va utiliser [flannel](https://github.com/flannel-io/flannel#flannel) :
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> Remarque : On utilise kubectl, cette commande nous permet d'effectuer des opérations sur le cluster depuis une machine distante, ici notre VM control ! Il est configuré grâce aux étapes précédentes. On notera aussi qu'il n'est pas nécessaire d'utiliser **sudo**.

On vérifie que tout fonctionne correctement avec ```kubectl get pods --all-namespaces```. On devrait voir plusieurs pods déployés, notamment un pod **flannel**.

### Nodes
> Pensez à désactiver le *swap* !

On peut maintenant ajouter les nodes au cluster. On va, pour chaque node, éxecuter la commande conservée précédemment. **Attention : la votre est différente de celle présentée en exemple ici, remplacez bien *xxx* par votre *hash* et modifiez le *token***
```shell
sudo kubeadm join 172.16.0.150:6443 --token 61z7u0.owjzoqjtzvm3djov --discovery-token-ca-cert-hash sha256:xxx
```
Notre cluster k8s est maintenant opérationnel, on peut désormais effectuer des opérations dessus via kubectl depuis *k8s-control*.

On peut avoir un aperçu du cluster avec la commande :
```shell
kubectl get nodes -o wide
```
### Installation d'un Load-Balancer
Nous allons installer un Load-Balancer au cluster, qui servira à exposezr nos services par la suite (en coopération avec un **ingress controler**). Nous utiliserons MetalLB.

> Nous suivons les instructions du site officiel de la solution : [MetalLB Installation](https://metallb.universe.tf/installation/#installation-by-manifest), [MetalLB Configuration](https://metallb.universe.tf/configuration/#layer-2-configuration).

Pour installer :
```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

Pour configurer (fichier : [metallb-conf.yaml](yaml/conf/metallb-conf.yaml)) :
```shell
kubectl create -f metallb-conf.yaml
```

> On peut tester la configuration comme suit :
> 
> ```shell
> kubectl create deploy nginx --image nginx
> kubectl expose deploy nginx --port 80 --type LoadBalancer
> kubectl get svc,pod
> ```
> 
> Le champ ```EXTERNAL-IP``` du service **nginx** doit correspondre à une addresse > de la plage référencée dans [metallb-conf.yaml](yaml/metallb-conf.yaml), et on > doit pouvoir afficher une page html sur cette ip :
> ```shell
> curl <EXTERNAL-IP>
> ```
> 
> Pour supprimer les ressources du test :
> ```shell
> kubectl delete svc nginx
> kubectl delete deploy nginx
> ```
> Notre Load-Balancer est opérationnel ! Vous pouvez déployer plusieurs pods/services selon la même méthode, une nouvelle addresse ip sera attribuée à chaque fois.

## Installation d'un fournisseur de stockage dynamique (avec NFS)
Un provisionement dynamique nous évite d'administrer manuellement les tailles de volumes au sein du Cluster. Ces espaces de stockages seront situés sur un serveur NFS distant, et les volumes persistants k8s seront de taille dynamique en fonction de la demande (création automatique d'un **persitent volume** lié à une partage **nfs** pour chaque déclaration de **persistent volume claim** utilisant notre **provisioner**).

### Serveur NFS
> Les actions usivantes sont à effectuer sur le serveur NFS distant, pas sur les nodes ni le controler du cluster k8s. Ici j'utilise un serveur linux CentOS 7, vous devrez adapter les commandes à votre distribution.

On commence par créer une règle Pare-Feu pour autoriser le service nfs à créer des partages :
```shell
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
**nfs** doit apparaître dans les services *(sortie de la dernière commande)*.

On installe les outils **nfs** :
```shell
sudo yum install nfs-utils
```

On renseigne le domaine dans ```/etc/idmapd.conf``` :
```conf
[General]
#Verbosity = 0
# The following should be set to the local NFSv4 domain name
# The default is the host's DNS domain name.
Domain = sound-buzz.live
(...)
```

On créé notre dossier à partager :
```shell
sudo mkdir /srv/nfs/kubedata -p
sudo chown nobody: /srv/nfs/kubedata
```
> La 2e commande permet d'éviter les problèmes de droits d'accès au partage par la suite.

On créé notre partage nfs dans le fichier ```/etc/exports``` :
```conf
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```
> Options de montage :
> - rw : permet de pouvoir lire et écrire sur le volume partagé
> - sync : permet d'appliquer les changements de façon immédiate sur le disque
> - no_subtree_check : On vérifie uniquement le **filesystem** du volume exporté, pas celui de l'arborescence complète
> - Empêche de mapper les utilisateurs *root* en *anonyme*
> - Même chose pour tous les utilisateurs (pas uniquement *root*)
> - Evite certaines restrictions sur les ports à employer

On peut maintenant démarrer et activer le service nfs :
```shell
sudo systemctl enable --now nfs-server rcpbind
sudo exportfs -rav
```

### Clients NFS sur le cluster
Pour chaque node sur le cluster (+ le control) :
```shell
sudo apt install nfs-common -y
```

> Pour tester que vous pouvez bien monter votre partage nfs sur chaque client :
> ```shell
> sudo mount -t nfs 172.16.0.8:/srv/nfs/kubedata /mnt
> ```
> 
> Vous pouvez ajouter/retirer des fichiers depuis le serveur nfs ou un autre client et observer les mêmes modifications depuis le client :
> ```shell
> ls -l /mnt
> ```
> 
> Vous pouvez aussi utiliser :
> ```shell
> mount | grep /mnt
> ```
> 
> Une fois que vous avez fini de vérifier, démontez le partage :
> ```shell
> sudo umount /mnt
> ```
> Vous pouvez tester sur chaque machine indépendemment, en simultané ou non.

### Déploiement du NFS-Provisioner
On va maintenant pouvoir déployer notre fournisseur sur le cluster (via la VM *k8s-control*).

> On se base sur ce projet : [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/v4.0.2).

Il existe plusieurs méthodes d'installation, nous allons utiliser **Helm**.

On commence par installer helm :
```shell
sudo snap install helm --classic
```

On installe ensuite le **repository** du projet :
```shell
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```
Puis on installe sur notre cluster :
```shell
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=172.16.0.8 --set nfs.path=/srv/nfs/kubedata
```

On peut utiliser ```helm list``` pour visualiser notre installation, et ```kubectl get sc``` pour vérifier que notre classe de stockage a bien été créée.

> Pour configurer cette installation et la personaliser à votre environement : [NFS Subdirectory External Provisioner Helm Chart](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/v4.0.2/charts/nfs-subdir-external-provisioner/README.md).

## Installation d'un contrôleur Ingress
> On va installer Traefik en tant que Ingress controler, en utilisant Helm : [traefik Install using the Helm chart](https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart).

Ajoutez le **repository** en suivant les instructions spécifiées dans [le lien précédent](https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart), puis éxecutez ces commandes :
```shell
helm show values traefik/traefik > traefik-values.yaml
sudo nano traefik-values.yaml
```

Modifiez la partie **persistence:** avec notre fournisseur nfs :
```yaml
persistence:
  enabled: true
  name: data
  accessMode: ReadWriteOnce
  size: 128Mi
  storageClass: "nfs-client"
  path: /data
  annotations: {}
```

Cela permettra de stocker nos certificats TLS de façon persistante, sans les perdre à chaque nouvelle instance de Traefik.

Pour installer Traefik avec notre configuration personnalisée :
```shell
helm install traefik traefik/traefik --values traefik-values.yaml
```

> En faisant ```kubectl get all```, on remarque que le service traefik utilise bien l'ip donnée par notre Load-Balancer MetalLB !

> Bonus : [Exposing Traefik Dashboard](https://doc.traefik.io/traefik/getting-started/install-traefik/#exposing-the-traefik-dashboard)  (fichier : [dashboards.yaml](yaml/dashboards/traefik-dashboard.yaml)).

## Utilisation de Let's Encrypt pour générer des certificats TLS
Afin d'utiliser le protocole https pour accéder à notre application, il nous faut des certificats TLS pour notre domaine. Le soucis c'est que depuis notre environement fermé esxi, nous ne pouvons pas réaliser de challenge http/https ou dns. On va donc émuler let's encrypt avec un server ACME local : [Pebble](https://github.com/letsencrypt/pebble).

> En production, il faudra remplacer Pebble par Let's Encrypt et adapter la procédure en conséquence.

Pour l'installation, nous allons utiliser Helm : [Pebble Helm Chart](https://github.com/jupyterhub/pebble-helm-chart).

On va éditer les valeurs du manifeste avant le déploiement :
```shell
helm show values jupyterhub/pebble > pebble-values.yaml
sudo nano pebble-values.yaml
```
Modifiez les parties **env:** et **coredns:** :
```yaml
 env:
  - name: PEBBLE_VA_ALWAYS_VALID
    value: "1"

 coredns:
   enabled: false
```
> La configuration de **env** avec ```PEBBLE_VA_ALWAYS_TRUE=1``` permet d'ignorer la validation d'identité par Pebble. C'est plus simple pour une preuve de concept comme ici.
> 
> **coredns** est optionel, on le désactive par souci de simplicité.

On déploie le serveur ACME :
```shell
helm install pebble jupyterhub/pebble --values pebble-values.yaml
```

On doit maintenant ajouter le certificat racine de Pebble à Traefik pour pouvoir effectuer un challenge https. On va pour cela utiliser la configMap mise à disposition par Pebble et mettre à jour les valeurs du manifeste **Helm** Traefik :
```yaml
volumes:
  - name: pebble
    mountPath: "/certs"
    type: configMap

#(...)

additionalArguments:
  - --certificatesresolvers.pebble.acme.tlschallenge=true
  - --certificatesresolvers.pebble.acme.email=test@sound-buzz.live
  - --certificatesresolvers.pebble.acme.storage=/data/acme.json
  - --certificatesresolvers.pebble.acme.caserver=https://pebble/dir

#(...)

env:
  - name: LEGO_CA_CERTIFICATES
    value: "/certs/root-cert.pem"
```

On applique nos changements :
```shell
helm upgrade --install traefik traefik/traefik --values traefik-values.yaml -n traefik
kubectl apply -f subsonic.yaml
```
> On vient d'importer le certificat root déliveré par pebble à notre serveur **front-end**. Celui-ci est automatiquement validé.
>
> Attention : ce certificat est déliveré par aue autorité privée, il n'est donc pas reconnu comme fiable par les navigateurs. Lorsque vous naviguerez sur les sites en https, il faudra valider ces certificats en tant que certificats de confiance.
>
> On évite ce problème en utilisant **Let's Encrypt**, et en effectuant la validation d'identité de notre serveur web.

## Déploiement de notre application
Maintenant que notre **front-end** est prêt (vous pouvez le tester avec les configurations du [poc nginx](yaml/POC/nginx-poc/)), nous allons déployer notre **back-end**.

On applique les configuartions de [sound-buzz.live](yaml/sound-buzz.live/) (fichier *all-in-one* : [3_subsonic-nfs.yaml](yaml\POC\subsonic-poc\3_subsonic-nfs.yaml)) :
```shell
kubectl apply -f <YAML_FILE>    # Pour créer/mettre à jour les éléments liés à la configuration
kubectl delete -f <YAML_FILE>   # Pour supprimer les éléments liés à la configuartion
```
> Remplacez **<YAML_FILE>** par le fichier de configuration à appliquer. Ils sont conçus pour être déployés dans cet ordre (et donc *delete* dans l'ordre inverse):
> - [storage.yaml](yaml/sound-buzz.live/storage.yaml) va créer les **claims** sur les volumes que l'on désire persistants en créant des **persistant volume** liés au partage **nfs** (à l'aide de notre **provisioner**).
> - [deployment.yaml](yaml/sound-buzz.live/deployment.yaml) va déployer l'application [subsonic](https://hub.docker.com/r/hurricane/subsonic) en utilisant nos volumes persistants.
  > > Dans un environement **k8s**, les conteneurs ne conservent pas les données crées au cours de leur utilisation lorsqu'ils sont recyclés. On doit donc créer des volumes externes afin de stocker ces données. Cela peut se faire comem ici à l'aide de volumes **nfs** paratgés, mais aussi à l'aide de connexions à des bases de données par exemple.
> - [service.yaml](yaml/sound-buzz.live/service.yaml) va créer un service exposant tous les pods de notre déploiement à une unique addresse ip (et un unique port).
> - [ingress.yaml](yaml/sound-buzz.live/ingress.yaml) va publier le service sur le web via une **ingressRoute**, redirigant au passage toutes les requêtes http vers https, et prenant en charge la gestion des certificats.

Pour aller plus loin : **autoscaling** *(option disponible avec Traefik directement)*, **renouvellement automatique des certificats** *(également avec Traefik, de concert avec Pebble/Let's Encrypt)*