# Deploy k3s cluster with helm and traefik, argoCD
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white) ![Rancher](https://img.shields.io/badge/rancher-%230075A8.svg?style=for-the-badge&logo=rancher&logoColor=white) ![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)

![Argo CD Status](https://img.shields.io/endpoint?url=https://argocd.dev.foot-idemia.fr/api/badge?name=idemiafoot-back)

 
## ArgoCD 
![Rancher](/assets/argocd.png)

## Prepare your VMs

-   Update your system
```shell
sudo apt update && sudo apt upgrade -y
```
-   Install dependencies
```shell
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

```
- Disable ufw, iptable (Enabled by default on OCI Cloud ubuntu instance)
```shell
sudo systemctl stop ufw.service 
sudo systemctl disable ufw.service 
sudo apt remove iptables-persistent
sudo ufw disable
```
to verify you can access the other vms, start simple python3 web server and try to curl it:
```shell
#On agent:
python3 -m http.server 9000
#From server:
curl -vv http://X.X.X.X.X:9000
```
Once time you can curl the server, network should be ok
### - Install helm: 
```bash
sudo snap install helm --classic
```

## Install k3s cluster

1. Install server from the install script provided by k3s team (link below)
https://docs.k3s.io/quick-start

There is the command with custom args: 
```bash
curl -sfL https://get.k3s.io/ | INSTALL_K3S_VERSION="v1.25.9+k3s1" INSTALL_K3S_EXEC="server --disable traefik --kubelet-arg=max-pods=500 --kubelet-arg=node-ip=10.0.0.13 --cluster-init --node-external-ip=129.151.228.19" sh

#Desciption:
--disable traefik #To disable the deployment of Traefik, a reverse proxy/load balancer.
--kubelet-arg=max-pods=500 #To set the maximum number of pods that can be created on the node to 500.
--kubelet-arg=node-ip=10.0.0.13 #To set the IP address of the node on which the Kubelet is running.
--cluster-init #To indicate that this node will be the first node in the cluster, and it should initialize a new cluster.
--node-external-ip=129.151.228.19 # To set the external IP address of the node.

```

3. Add right to k3s.yaml
```bash
sudo chmod 755 /etc/rancher/k3s/k3s.yaml
```

4. Add helm repo :
- jetstack
- traefik
- rancher
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo add traefik https://traefik.github.io/charts
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

5. Install cert-manager with helm
```bash
helm upgrade --install cert-manager wjetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true --set 'extraArgs={--acme-http01-solver-nameservers=10.43.0.93:53}' --set podDnsPolicy=None --set podDnsConfig.nameservers={'10.43.0.93}'
```
6. Install Traefik with helm
Simple traefik install using helm 
```bash
#Install and add aditionnal values:
helm install --namespace traefik --create-namespace traefik traefik/traefik --values=traefik/values.yml
#Apply traefik http/https middleware
k apply -n traefik -f traefik/middleware-http-https.yml
```

7. Install Rancher
```bash
helm install rancher rancher-latest/rancher   --namespace cattle-system  --create-namespace  --set hostname=rancher.foot-idemia.fr  --set replicas=1   --set ingress.tls.source=letsEncrypt   --set letsEncrypt.email=loic.verniere@icloud.com   --set letsEncrypt.ingress.class=traefik   --set global.cattle.psp.enabled=false
```
8. Deploy cluster secret
```bash
k apply -f cluster-secret && k apply -f cluster-secret 
```
9. If you want an other master (with or without etcd):
For a master without etcd (state database), for multiple etcd remove --disable-etcd, multiple etcd is better for fault tolerance !! But in cas of network unavailability it may corrupt both etcd

From the master vm (get the join token):
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

From the vm you want to join the cluster:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.25.9+k3s1" K3S_TOKEN=TOKEN sh -s - server --disable traefik --disable-etcd --kubelet-arg=max-pods=500 --server https://10.0.0.13:6443

# remove if you want etcd on the other master:
--disable-etcd
# Add master Ip
--server 
```

For a worker:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.25.9+k3s1" K3S_URL=https://<SERVER IP TO JOIN>:6443 INSTALL_K3S_EXEC="--kubelet-arg=max-pods=500" K3S_TOKEN=TOKEN sh -
```

## Install Longhorn

You shoul be able to access rancher from the the dns you specified during the install, for me: rancher.foot-idemia.fr

Click on nodes
![Rancher](/assets/rancher.png)

Now install longhorn

How it works
![longhorn](/assets/how-longhorn-works.svg)


