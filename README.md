# nextcloud-microk8s
For microk8s single node cluster installation see [official documentation](https://microk8s.io/docs)

## Prepare microk8s before first deployment

### Enable microk8s DNS addon
Microk8s provide a DNS addon (CoreDNS) to setup a DNS server for your cluster.
Other addons like Storage cause some errors or not functioning well without a cluster DNS.

``` shell
microk8s enable dns
```

Wait until CoreDNS is running
``` shell
kubectl get pods -n kube-system --watch
```

``` shell
NAME                         READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-xxxxx     1/1     Running   0          69s
```

### Enable microk8s Storage addon
Microk8s provides a storage addon which persist data on your local node filesystem.

``` shell
microk8s enable storage
```

Wait until `hostpath-provisioner` is running
``` shell
kubectl get pods -n kube-system --watch
```

``` shell
NAME                                      READY   STATUS    RESTARTS   AGE
hostpath-provisioner-xxxxxxxxxx-xxxxx     1/1     Running   0          23m
```


### Enable microk8s Ingress addon
Mircok8s provides with the addon `ingress` an nginx ingress controller implementation.

``` shell
microk8s enable ingress
```

Wait until your ingress controller is ready

``` shell
kubectl get pods -n ingress --watch
```

``` shell
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-xxxxx   1/1     Running   0          79s
```


### Enable microk8s MetalLB addon (Optional for TLS)
In a self-managed context you need a load balancer for communication
with Let´s Encrypt. In this case we use microk8´s metallb addon.

Install metallb addon with the `Public IP` of your host.

``` shell
microk8s enable metallb:xxx.xxx.xxx.xxx-xxx.xxx.xxx.xxx
```


### Deploy Cert-manager (Optional for TLS)
[Cert-manager](https://cert-manager.io/docs/) adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates. It will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

In this example we use `Cert-manager` together with [Let´s encrypt](https://letsencrypt.org/de/docs/). 

Install latest version of [Cert-Manager](https://github.com/jetstack/cert-manager/tags) for let´s encrypt certificates.

``` shell
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```  

Verify if ```cert-manager``` , ```cainjector ``` and ```webhook``` are running

``` shell
kubectl get pods --namespace cert-manager --watch
``` 

``` shell
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-cainjector-xxxxxxxxxx-xxxxx   1/1     Running   0          19s
cert-manager-xxxxxxxxxx-xxxxx              1/1     Running   0          19s
cert-manager-webhook-xxxxxxxxxx-xxxxx      1/1     Running   0          19s
``` 


## Deploy nextcloud-k8s

### Create namespace nextcloud

Change your namespace from using `default` to `nextcloud`.
For all resources for which no explicit namespace was set, are now created in namespace `nextcloud` 

``` shell
kubectl create namespace nextcloud
```

``` shell
kubectl config set-context --current --namespace=nextcloud
```


### Create deployment secrets


``` shell
kubectl create secret generic nextcloud-secrets \
    --from-literal=MYSQL_ROOT_PASSWORD=CHANGE_ME \
    --from-literal=MYSQL_USER=CHANGE_ME \
    --from-literal=MYSQL_PASSWORD=CHANGE_ME \
    --from-literal=NEXTCLOUD_ADMIN_USER=CHANGE_ME \
    --from-literal=NEXTCLOUD_ADMIN_PASSWORD=CHANGE_ME
```
### Create deployment configMap

``` yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextcloud-configmap
data:
  MYSQL_DATABASE: CHANGE_ME
  MYSQL_HOST: CHANGE_ME
  NEXTCLOUD_TRUSTED_DOMAINS: CHANGE_ME
  ADMINER_DESIGN: pepa-linha
  ADMINER_DEFAULT_SERVER: CHANGE_ME (same as MYSQL_HOST)
EOF
```

### Deploy Let´s Encrypt issuer production (Optional for TLS)
To make a request to Let´s Encrypt API you have to create an `ìssuer` object first.
Please change `email: user@example.com` before you run the following command.

``` yaml
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod-issuer
spec:
  acme:
    # The ACME production server
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod-issuer
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class:  public
EOF
```

### Create deployment PVC
With this PersistentVolumesClaim(PVC) we request a storage of 10Gi from our PersistenVolume(PV)
provided by microk8s´s Storage addon.

``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-k8s/main/nextcloud-pvc.yaml
```

Check if your PVC is bound properly

``` shell
kubectl get pvc nextcloud-shared-storage-claim
``` 

``` shell
NAME                             STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS        AGE
nextcloud-shared-storage-claim   Bound    pvc-xxx   10Gi       RWO            microk8s-hostpath   9s
``` 


### Create deployment nextcloud-db
Deploy a mariadb database as backend for nextcloud-server

``` shell
kubectl apply -f  https://raw.githubusercontent.com/kinkaraCoding/nextcloud-k8s/main/nextcloud-db.yaml
```

### Create deployment nextcloud-server
Deploy nextcloud frontend server


``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-k8s/main/nextcloud-server.yaml
```  


### Create deployment nextcloud-adminer
Deploy PHP based web ui for nextcloud-db
``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-k8s/main/nextcloud-adminer.yaml
```  


## Create Ingress
Put your correct domain in both host sections for `host: nextcloud.example.com`and `host: adminer.example.com`

``` shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress
spec:
  rules:
  - host: nextcloud.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nextcloud-server
          servicePort: 80
  - host: adminer.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nextcloud-server
          servicePort: 8080
EOF
```  




## Create Ingress with TLS 
Put your correct domain in both host sections for `host: nextcloud.example.com`and `host: adminer.example.com`

``` shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress-prod-tls
  annotations:
    kubernetes.io/ingress.class: "public"    
    cert-manager.io/issuer: "letsencrypt-prod-issuer"
spec:
  tls:
  - hosts:
    - nextcloud.example.com
    - adminer.example.com
    secretName: letsencrypt-prod-cert
  rules:
  - host: nextcloud.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nextcloud-server
            port:
              number: 80
  - host: adminer.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nextcloud-adminer
            port:
              number: 8080
EOF
```

Verify if certificate named `letsencrypt-prod-cert` is ready. This process can take up to 2 minutes

```Shell
kubectl get certificate letsencrypt-prod-cert --watch
```

```Shell
NAME                   READY   SECRET                 AGE
nextcloud-secret-tls   True    nextcloud-secret-tls   13s
```
