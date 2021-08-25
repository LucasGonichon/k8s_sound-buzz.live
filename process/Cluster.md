## Création du cluster
### Controler
Avant de commencer, on désactive le *swap* sur la VM *k8s-controler* : ```sudo swapoff -a```

Uniquement sur la VM *k8s-controler*, on execute la commande suivante :
```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

La commande devrait sortir dans le terminal une ligne de ce type :
```kubeadm join 172.16.0.150:6443 --token 61z7u0.owjzoqjtzvm3djov --discovery-token-ca-cert-hash sha256:0200952490e9f8a8834c62d7545400a652f82674f0bad35663d1acd01ac623b7```.

Gardez cette commande de côté, elle servira plus tard.
> Attention : Ce sont des données assez sensibles d'un point de vue sécurité, faites attention à ne pas partager cette commande, et à ne pas la conserver une fois qu'elle n'est plus utile.

Lancez maintenant les commandes suivantes :
```shell
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Elles se trouvent aussi dans la sortie de la commande précédente, si elles sont différentes chez vous, adaptez cette étape.

On doit maintenant installer un driver pour gérer le réseau au sein du cluster. On va utiliser [flannel](https://github.com/flannel-io/flannel#flannel) :
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> Remarque : On utilise kubectl, cette commande nous permet d'effectuer des opérations sur le cluster depuis une machine distante, ici notre VM control ! Il est configuré grâce aux étapes précédentes. On notera aussi qu'il n'est pas nécessaire d'utiliser **sudo**.

On vérifie que tout fonctionne correctement avec ```kubectl get pods --all-namespaces```. On devrait voir plusieurs pods déployés, notamment un pod **flannel**.

### Nodes

> Pensez à désactiver *swap* pour les nodes (de la même manière que pour le controler) avant de join le cluster.

On peut maintenant ajouter les nodes au cluster. On va, pour chaque node, éxecuter la commande conservée précédemment (avec l'ajout de sudo cette fois ci ; attention la votre est différente de celle présentée en exemple ici).

```shell
sudo kubeadm join 172.16.0.150:6443 --token 61z7u0.owjzoqjtzvm3djov --discovery-token-ca-cert-hash sha256:0200952490e9f8a8834c62d7545400a652f82674f0bad35663d1acd01ac623b7
```

Notre cluster k8s est maintenant opérationnel, on peut désormais effectuer des opérations dessus via kubectl depuis *k8s-control*.

On peut avoir un aperçu du cluster avec la commande :
```shell
kubectl get nodes -o wide
```
## Installation d'un Load-Balancer
Nous allons installer un Load-Balancer au cluster afin de répartir la charge sur les différents nodes. Nous utiliseront MetalLB.

> Nous suivons les instructions du site officiel de la solution : [MetalLB Installation](https://metallb.universe.tf/installation/#installation-by-manifest), [MetalLB Configuration](https://metallb.universe.tf/configuration/#layer-2-configuration).

Pour installer :
```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

Pour configurer (fichier : [metallb-conf.yaml](yaml/metallb-conf.yaml)) :
```shell
kubectl create -f metallb-conf.yaml
```

On peut tester la configuration comme suit :

```shell
kubectl create deploy nginx --image nginx
kubectl expose deploy nginx --port 80 --type LoadBalancer
kubectl get svc,pod
```

Le champ ```EXTERNAL-IP``` du service **nginx** doit correspondre à une addresse de la plage référencée dans [metallb-conf.yaml](yaml/metallb-conf.yaml), et on doit pouvoir afficher une page html depuis le pod nginx via le service en faisant :
```shell
curl 172.16.0.200
```

Pour supprimer les ressources du test :
```shell
kubectl delete svc nginx
kubectl delete deploy nginx
```