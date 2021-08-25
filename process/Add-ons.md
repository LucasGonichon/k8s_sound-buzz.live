
## Installation d'un fournisseur de stockage dynamique (avec NFS)
Un provisionement dynamique nous évite d'administrer manuellement les tailles de volumes au sein du Cluster. Ces espaces de stockages seront situés sur un serveur NFS distant, et les volumes persistants k8s seront de taille dynamique en fonction de la demande (ajout de musique, de certificats TLS, etc).

### Serveur NFS
> Les actions usivantes sont à effectuer sur le serveur NFS distant, pas sur les nodes ni le controler du cluster k8s. Ici j'utilise un serveur linux CentOS 7, vous devrez adapter les commandes à votre distribution.

On commence par créer une règle Pare-Feu pour autoriser le service nfs :
```shell
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
**nfs** doit apparaître dans les services.

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

On créé notre partage nfs dans le fichier ```/etc/exports``` :
```conf
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

On peut maintenant démarrer et activer le service nfs :
```shell
sudo systemctl enable --now nfs-server rcpbind
sudo exportfs -rav
```

### Clients NFS sur le cluster
Pour chaque node sur le cluster (et le control) :
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

### Déploiement du NFS-Provisioner
On va maintenant pouvoir déployer notre fournisseur sur le cluster (via *k8s-control*).

> On se base sur ce projet : [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/v4.0.2).

Il existe plusieurs méthodes d'installation, nous allons utiliser **Helm**.

On commence par installer helm :
```shell
sudo snap install helm --classic
```

On installe ensuite le **repository** :
```shell
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```
Puis on installe le projet sur notre cluster :
```shell
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=172.16.0.8 --set nfs.path=/srv/nfs/kubedata
```

On peut utiliser ```helm list``` pour visualiser notre installation, et ```kubectl get sc``` pour vérifier que notre classe de stockage a bien été créée.

> Pour configurer cette installation et la personaliser à votre environement : [NFS Subdirectory External Provisioner Helm Chart](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/v4.0.2/charts/nfs-subdir-external-provisioner/README.md).

## Installation d'un contrôleur Ingress
> On va installer Traefik en tant que Ingress controler, en utilisant Helm : [traefik Install using the Helm chart](https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart).

Ajoutez le **repository** en suivant les instructions, puis éxecutez ces commandes :
```shell
helm show values traefik/traefik > traefik-values.yaml
sudo nano traefik-values.yaml
```

Modifiez la partie **persistence** avec notre fournisseur nfs :
```yaml
persistence:
  enabled: true
  name: data
#  existingClaim: ""
  accessMode: ReadWriteOnce
  size: 128Mi
  storageClass: "nfs-client"
  path: /data
  annotations: {}
  # subPath: "" # only mount a subpath of the Volume into the pod
```

Cela permettra de stocker nos certificats TLS de façon persistante, sans les perdre à chaque nouvelle instance de Traefik. Conservez ce fichier, nous le modifieront à nouveau par la suite (pour l'autoscaling notamment).

pour installer Traefik avec notre configuration personnalisée :
```shell
helm install traefik traefik/traefik --values traefik-values.yaml -n traefik --create-namespace
```

> En faisant ```kubectl get all -n traefik```, on remarque que le service traefik utilise bien l'ip donnée par notre Load-Balancer MetalLB !

> Bonus : [Exposing Traefik Dashboard](https://doc.traefik.io/traefik/getting-started/install-traefik/#exposing-the-traefik-dashboard)  (fichier personnalisé : [dashboards.yaml](yaml/dashboards.yaml)).

## Déploiement de l'application
On est maintenant prêts à déployer notre application. Nous allons utiliser l'application subsonic en utilisant l'image suivante : [hurricane/subsonic](https://hub.docker.com/r/hurricane/subsonic).

On commence par créer un enregistrement DNS sur l'ip publique de notre load-balancer (enregistrement A vers ```172.16.0.200```)

Nous pouvons simplement appliquer la configuration [subsonic.yaml](yaml/subsonic.yaml) :
```shell
kubectl apply -f subsonic.yaml
```

On peut maintenant accéder à l'app subsonic depuis **app.subsonic.live**.

> Côté backend, les données de l'app sont persistantes sur des volumes *nfs* auto-provisionnés.

## Utilisation de Let's Encrypt pour générer des certificats TLS
Afin d'utiliser le protocole https pour accéder à notre application, il nous faut des certificats TLS pour notre domaine. Le soucis c'est que depuis notre environement fermé esxi, nous ne pouvons pas réaliser de challenge http/https ou dns. On va donc émuler let's encrypt avec un server ACME local : [Pebble](https://github.com/letsencrypt/pebble).

> En production, il faudra remplacer Pebble par Let's Encrypt et adapter la procédure en conséquence.

Pour l'installation, nous allons utiliser Helm : [Pebble Helm Chart](https://github.com/jupyterhub/pebble-helm-chart).

On va éditer les valeurs du manifeste avant le déploiement :
```shell
helm show values jupyterhub/pebble > pebble-values.yaml
sudo nano pebble-values.yaml
```
Modifiez les parties **env:** **coredns:** :
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

On déploie le serveur ACME dans le même *namespace* que Traefik par soucis de simplicité.
```shell
helm install pebble jupyterhub/pebble --values pebble-values.yaml -n traefik
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

On modifie notre ingress pour subsonic pour tester :
```yaml
entrypoints:
  - websecure

#(...)

tls:
  certResolver: pebble
```
On applique nos changements :
```shell
helm upgrade --install traefik traefik/traefik --values traefik-values.yaml -n traefik
kubectl apply -f subsonic.yaml
```

On peut maintenant accéder à https://app.sound-buzz.live au lieu de http://sound-buzz.live.

> En http, on a une erreur 404 : ingress ne prend plus en compte les requêtes http (plus de entrypoint *web* : remplacé par *websecure*).
>
> En https: il faut accepter le certificat dans le navigateur : celui-ci n'est pas délivré par une autorité fiable.

On peut mettre en place une redirection automatique de http vers https en appliquant [redirect.yaml](yaml/redirect.yam).