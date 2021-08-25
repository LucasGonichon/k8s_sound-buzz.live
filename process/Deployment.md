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