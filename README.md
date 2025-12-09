
Reference : 
https://medium.com/@alex.ivenin/how-to-run-kubernetes-cluster-with-k3s-a815da0b70f9

## Always useful

https://github.com/alexellis/k3sup

Get a Ubuntu bash console within the cluster : `kubectl run my-shell --rm -i --tty --image ubuntu -- bash`

## On master node

```sh
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/


# sudo visudo

# Then add to the bottom of the file
# replace "alexo" with your username i.e. "ubuntu"
alexo ALL=(ALL) NOPASSWD: ALL


k3sup install --user alexo --local --k3s-extra-args '--disable servicelb'



# curl -sfL https://get.k3s.io | sh -s - --disable=servicelb

# sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
# sudo chown alexo.alexo ~/.kube/config
# sudo chmod o+r /etc/rancher/k3s/k3s.yaml


watch sudo k3s kubectl get pods -A

# cat ~/.kube/config

# sudo vi /etc/multipath.conf
# blacklist {
#     devnode "^sd[a-z0-9]+"
# }
# systemctl restart multipathd.service

sudo cat /var/lib/rancher/k3s/server/node-token
```


## On worker nodes
Get the token from the master node with `sudo cat /var/lib/rancher/k3s/server/node-token`

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://example.com:6443 sh -s - agent --token <token>

watch sudo k3s kubectl get pods -A

sudo vi /etc/multipath.conf
blacklist {
    devnode "^sd[a-z0-9]+"
}
systemctl restart multipathd.service
sudo chmod o+r /etc/rancher/k3s/k3s.yaml
```

## Setuping the various services

`kubectl label node nodename1 node-role.kubernetes.io/worker=worker`

`kubectl label node nodename2 node-role.kubernetes.io/worker=worker`

`argocd account update-password --account alexo`


### longhorn

Install open-iscsi

```sh
sudo apt-get install open-iscsi
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/prerequisite/longhorn-iscsi-installation.yaml

kubectl get pod | grep longhorn-iscsi-installation
kubectl logs longhorn-iscsi-installation-<POD> -c iscsi-installation

apt-get install nfs-common
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/prerequisite/longhorn-nfs-installation.yaml


kubectl apply -f longhorn.yaml
kubectl apply -f ingress.yaml

kubectl delete sc local-path
```

### ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


### kustomize

For use with kustomize and ArgoCD, I highly suggest that you fork this repo into a private repo, and create your own overlay with your own configs specific to your environment.  The idea is that you want to separate your environment from your declaration of an environment.  Hence, you will have a "base" infrastructure repo, and an "overlay" repo.  The overlay repo will contain all of the configs specific to your environment(s), and the "base" will only contain what you want in this environment, regardless of how it is configured.

It is meant to be used with ArgoCD.  I highly suggest installing ArgoCD, pointing it to your overlay repo for your environment, and then letting it run everything from there.