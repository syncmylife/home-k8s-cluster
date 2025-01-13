
# Target: Kubernetes on Ubuntu LTS with microk8s using Longhorn for persistent storage. v20250112
### [#k8s #ubuntuLTS #microk8s #helm #metallb #cert-manager #longhorn #ingress #letsencrypt #rancher #dashboard #reflector #longhorn #gitea #nextcloud #rsync #jupyterhub #lms #rabbitmq  #smartmontools #smartd #influxdb #grafana #nodered #home-assistant #docker]

## FAQ:
### Why kubernetes?
: best standard tech. solution for 2 and more nodes with distributed storage
### Why Ubuntu LTS?
: debian packaging system with LTS support
### Why microk8s?
: simple and on your machine
### Why distributed storage?
: support for two data locations
### Why longhorn?
: because there is/was no better alternative
### Why wildcard certificate?
: securing both primary and subdomains with a single certificate
### Known limitations?
: loadBalancer metallb only on the master node = no full redundancy

## Hardware

### main node (location 1)
- MIKROTIK RB5009UG+S+IN
- Desktop PC with Ryzen 5600g, 64GB RAM, 1TB SSD, 14TB HDD(crypted lvm)
### secondary node (location 2)
- MIKROTIK RBD52G-5HacD2HnD-TC
- HP ProLiant MicroServer Gen8, Xeon e3-1270, 8GB RAM, 1TB SSD, 8TB+6TB HDD(crypted lvm)

## Network
* both nodes are connected using wireguard 
* node1 has the hostname amber.<example.cloud> LAN IP 192.168.201.10 and node2 vg.<example.cloud> 192.168.101.235
* both nodes have in router defined primary DNS 8.8.8.8 but with static DNS entries: \
  amber.<example.cloud> 192.168.210.10 \
  vg.<example.cloud> 192.168.101.235 \
  git.<example.cloud> 192.168.210.222 \
  <example.cloud> 192.168.210.200 \
  .+\.<example>\.<cloud> 192.168.210.200
* both nodes are running Ubuntu 24.04 LTS with microk8s 1.32/stable and user cuser

## Installing extra packages

```console
cuser@amber:~$ sudo apt install mc
cuser@amber:~$ sudo apt-get install net-tools
cuser@amber:~$ ifconfig
```

```console
cuser@vg:~$ sudo apt install mc
cuser@vg:~$ sudo apt-get install net-tools
cuser@vg:~$ ifconfig
```

## Configure hostnames
(optional: this is also possible with: sudo hostnamectl set-hostname)

```console
cuser@amber:~$ sudo vim /etc/hostname
amber.<example.cloud>

cuser@amber:~$ sudo reboot
```

```console
cuser@amber:~$ sudo vim /etc/hosts
```

* add this lines to /etc/hosts:

```console
127.0.1.1 amber.<example.cloud>
192.168.101.235 vg.<example.cloud>
```

* now the same for the other node:

```console
cuser@vg:~$ sudo vim /etc/hostname
vg.<example.cloud>

cuser@vg:~$ sudo reboot
```

```console
cuser@vg:~$ sudo vim /etc/hosts
```

* add this lines to /etc/hosts:

```console
127.0.1.1 vg.<example.cloud>
192.168.210.10 amber.<example.cloud>
```

* on both of the mikrotik router allow remote requests(do not forget the firewall rules forbidding port 53 TCP and UDP from WAN) this way:
```console
name: <example.cloud> to 192.168.210.200
regexp: .+\.<example>\.<cloud> to 192.168.210.200
```
* flush DNS on mikrotik
* flush DNS on ubuntu:
```console
cuser@amber:~$ sudo resolvectl flush-caches
cuser@vg:~$ sudo resolvectl flush-caches
```
* flush DNS on windows PCs:
```console
ipconfig /flushdns
```
* try cuser@amber:~$ dig <example.cloud> .. if the response contains the public IP then then DNS is not set to the router

* verifying that DNS is working correctly within your Kubernetes platform: https://help.hcl-software.com/connections/v6/admin/install/cp_prereq_kubernetes_dns.html

## Configure DNS with systemd-resolved
* https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
* we want to set our router as a dns so that we can use static entries in out router for example for git.<example.cloud>
* check if resolv.conf is a symlink to /run/systemd/resolve/stub-resolv.conf:
```console
cuser@amber:~$ ls -l /etc/resolv.conf
cuser@vg:~$ ls -l /etc/resolv.conf
```

* change the /etc/systemd/resolved.conf for both nodes:
$ sudo vim /etc/systemd/resolved.conf
DNS=192.168.210.1
FallbackDNS=8.8.8.8

* restart to apply the configuration:
```console
cuser@amber:~$ sudo service systemd-resolved restart
cuser@vg:~$ sudo service systemd-resolved restart
```

* check the current output for DNS:
```console
cuser@amber:~$ resolvectl
cuser@vg:~$ resolvectl
```

* check if we can resolve the IP for git.<example.cloud>
```console
cuser@amber:~$ dig git.xsync.cloud
cuser@amber:~$ dig git.xsync.cloud
```

## Update system packages(for all nodes)
```console
cuser@amber:~$ sudo apt update && sudo apt upgrade -y
```

```console
cuser@vg:~$ sudo apt update && sudo apt upgrade -y
```

## Configure ssh
* I don't have a ~/.ssh folder, so:
```console
cuser@amber:~$ cd ~
cuser@amber:~$ ssh-keygen
```
* when asked provide a {replace-this-with-your-passphraze}
 
```console
cuser@vg:~$ cd ~
cuser@vg:~$ ssh-keygen {replace-this-with-your-passphraze}
```

* then from the primary server(the IP/hostname is the secondary server):
```console
cuser@amber:~$ ssh-copy-id cuser@vg.<example.cloud>
```

* provide the passphraze and test with:
```console
cuser@amber:~$ ssh 'cuser@vg.<example.cloud>'
```

```console
cuser@vg:~$ ssh 'cuser@amber.<example.cloud>'
```

* do not forget that my firewall has a default rule to drop connections coming from 192.168.101.0 so modify this

## Creating a fast storage (on each node)
* create and mount fast storage - /mnt/ssd has to be available on all nodes(each node has to provide 700G!)

```console
cuser@amber:~$ sudo lvcreate -n ssd -L 700G ubuntu-vg
cuser@amber:~$ sudo mkfs.ext4 /dev/ubuntu-vg/ssd
cuser@amber:~$ sudo mkdir /mnt/ssd
cuser@amber:~$ sudo blkid /dev/ubuntu-vg/ssd
```

* if the output of blkid is empty then restart the pc and try again
```console
cuser@amber:~$ 7db13169-52bd-4bdb-b6bc-bcc627a4afde
cuser@amber:~$ sudo vim /etc/fstab
```

* add this lines to /etc/fstab:

```console
# /mnt/ssd
/dev/disk/by-uuid/7db13169-52bd-4bdb-b6bc-bcc627a4afde /mnt/ssd ext4 defaults 0 1
```

* save changes and remount:

```console
cuser@amber:~$ sudo mount -a
```

* for the vg node do the same: create and mount fast storage - /mnt/ssd has to be available on all nodes(each node has to provide 700G!)

```console
cuser@vg:~$ sudo lvcreate -n ssd -L 700G ubuntu-vg
cuser@vg:~$ sudo mkfs.ext4 /dev/ubuntu-vg/ssd
cuser@vg:~$ sudo mkdir /mnt/ssd
cuser@vg:~$ sudo blkid /dev/ubuntu-vg/ssd
```

* if the output of blkid is empty then restart the pc and try again
```console
cuser@vg:~$ d6e7bdc3-d0e0-4c71-944e-cc6792cc6a00
cuser@vg:~$ sudo vim /etc/fstab
```

* add this lines to /etc/fstab:

```console
# /mnt/ssd
/dev/disk/by-uuid/d6e7bdc3-d0e0-4c71-944e-cc6792cc6a00 /mnt/ssd ext4 defaults 0 1
```

* save changes and remount:

```console
cuser@vg:~$ sudo mount -a
```

* install cryptsetup and dmsetup:
```console
cuser@vg:~$ sudo apt-get install cryptsetup
cuser@vg:~$ sudo apt-get install dmsetup

cuser@amber:~$ sudo apt-get install cryptsetup
cuser@amber:~$ sudo apt-get install dmsetup
```

## Mounting crypted HDDs(version for secondary node with 2 crypted HDD's)
```console
cuser@vg:~$ cat /proc/partitions
cuser@vg:~$ sudo blkid /dev/sdb
UUID="67e52429-1daa-4c96-95f7-716f0193826e" TYPE="crypto_LUKS"
cuser@vg:~$ sudo blkid /dev/sdc
UUID="e9834bd0-e366-46fa-80e5-7db71bc01908" TYPE="crypto_LUKS"
```
* create mount-fileserver-data.sh script somewhere:

```console
sudo touch /media/mount-fileserver-data.sh
sudo chown cuser:cuser /media/mount-fileserver-data.sh
sudo chmod +x /media/mount-fileserver-data.sh
```

* this is the content of the mount-fileserver-data.sh script:

```console
cuser@vg:~$ cat mount-fileserver-data.sh

#!/bin/bash
if [[ $(id -u) -ne 0 ]] ; then echo "Please run as root with: sudo ./mount-fileserver-data.sh" ; exit 1 ; fi
# Read Password
echo -n Please enter the cryptsetup password for sda:
read -s password
echo
# Run Command
echo $password | cryptsetup luksOpen /dev/disk/by-uuid/67e52429-1daa-4c96-95f7-716f0193826e crypted_sdb
echo $password | cryptsetup luksOpen /dev/disk/by-uuid/e9834bd0-e366-46fa-80e5-7db71bc01908 crypted_sdc
lvscan
mount /dev/vgb01/lv00_main /mnt/8tb
mount /dev/vg1/lv_data /mnt/6tb
```
## Install microk8s (if not already installed)
* As I have installed ubuntu server with snapd (and microk8s) !I DONT need this!:
```console
cuser@amber:~$ sudo apt update
cuser@amber:~$ sudo apt install snapd
cuser@amber:~$ sudo snap install microk8s --classic --channel=1.31/stable
```
## Connecting nodes
* on the main node:
```console
cuser@amber:~$ microk8s add-node
```
* copy the join command from the output of the add-node command, its sth. like:
```console
microk8s join 192.168.x.x:25000/73d4fs456452656vh6fdbv7vsda8bg52/6d7fj6456j94 --worker
```

and run it:
```console
cuser@vg:~$ microk8s join 192.168.x.x:25000/73d4fs456452656vh6fdbv7vsda8bg52/6d7fj6456j94 --worker
cuser@amber:~$ microk8s kubectl get no
```
* (when worker successfully joined the master, but 
```console
kubectl get nodes
```
is not returning the worker then the part with --worker was missing) 
* when something went wrong its easy possible to debug and remove the whole microk8s:
```console
cuser@amber:~$ journalctl -u snap.microk8s.daemon-kubelite.service -n 1000
cuser@amber:~$ sudo  rm -rf /usr/local/bin/kubectl
cuser@amber:~$ sudo snap remove --purge microk8s
cuser@amber:~$ sudo rm -rf /var/lib/snapd/cache/*
cuser@amber:~$ sudo rm -rf ~/.kube/*
```

* sometimes the microk8s is stopped and throws an error(and not on every command when the problem is not on the master):
Get "https://127.0.0.1:16443/version": dial tcp 127.0.0.1:16443: connect: connection refused
or
The connection to the server 127.0.0.1:16443 was refused - did you specify the right host or port?

```console
cuser@vg:~$ microk8s.status
cuser@vg:~$ microk8s.start
cuser@vg:~$ microk8s.status
```

## Configure microk8s, enable hostpath-storage, dns
* call with the right admin user(cuser)
```console
cuser@amber:~$ sudo usermod -a -G microk8s $USER
cuser@amber:~$ sudo chown -f -R $USER ~/.kube
cuser@amber:~$ newgrp microk8s
```

* check the channel info and change to the desired channel:
```console
cuser@amber:~$ snap info microk8s

cuser@amber:~$ sudo snap refresh microk8s --channel=1.31/stable
```

* to debug microk8s use:
```console
cuser@amber:~$ journalctl -u snap.microk8s.daemon-kubelite.service -n 1000
```

* change the config:

```console
cuser@amber:~$ mkdir -p ~/.kube
cuser@amber:~$ microk8s kubectl config view --raw >~/.kube/config
cuser@amber:~$ chmod 600 ~/.kube/config

cuser@amber:~$ sudo snap alias microk8s.kubectl kubectl
	
cuser@amber:~$ kubectl get nodes -o wide --show-labels
cuser@amber:~$ kubectl label nodes amber.<example.cloud> kubernetes.io/role=master
cuser@amber:~$ kubectl label nodes amber.<example.cloud> disktype=ssd
cuser@amber:~$ kubectl label nodes vg.<example.cloud> disktype=ssd
cuser@amber:~$ kubectl label nodes amber.<example.cloud> storagetype=fileserver

cuser@amber:~$ sudo microk8s enable hostpath-storage dns
```
## Install helm
* as I like to administer helm by snap and not by microk8s:

```console
cuser@amber:~$ sudo microk8s disable helm
cuser@amber:~$ sudo snap install helm --classic
```

* uninstalling helm if needed:
```console
cuser@amber:~$ helm uninstall cert-manager -n cert-manager
```

## Install cert-manager
* latest compatible version with k8s is listed under: https://cert-manager.io/docs/releases/
* you can use https://github.com/cert-manager/cert-manager/releases/latest to list the latest stable tag (here v1.11.1)

```console
cuser@amber:~$ helm repo add jetstack https://charts.jetstack.io
cuser@amber:~$ helm repo update


* we need the dns01-recursive-nameservers flag because by default we have our own router as dns
cuser@amber:~$ helm install \
cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.16.2 \
--set crds.enabled=true \
--set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}'
```
* check for running pods:

```console
cuser@amber:~$ kubectl get pods --namespace cert-manager
```

## Install metallb
* install with helm and disable metallb - we want to have it in a separate namespace

```console
cuser@amber:~$ microk8s disable metallb
cuser@amber:~$ helm repo add metallb https://metallb.github.io/metallb
cuser@amber:~$ helm repo update
	
cuser@amber:~$ helm install metallb metallb/metallb \
--namespace metallb-system \
--create-namespace \
--version 0.14.9

cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.210.200-192.168.210.254
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-ip
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

* does it work?

```console
cuser@amber:~$ kubectl get all -n metallb-system
cuser@amber:~$ kubectl get ipaddresspools.metallb.io -n metallb-system
cuser@amber:~$ kubectl get l2advertisements.metallb.io -n metallb-system
cuser@amber:~$ kubectl describe ipaddresspools.metallb.io default-pool -n metallb-system

cuser@amber:~$ kubectl create deployment nginx --image=nginx
cuser@amber:~$ kubectl get pods
cuser@amber:~$ kubectl expose deployment nginx --type=LoadBalancer --port 80
cuser@amber:~$ kubectl get services
```

```console
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.43.0.1       <none>          443/TCP        5h53m
nginx        LoadBalancer   10.43.156.110   192.168.28.11   80:32580/TCP   117s
```

* To access the load-balancer type service, we need to retrieve the EXTERNAL-IP of the nginx-server:

```console
cuser@amber:~$ INGRESS_EXTERNAL_IP=`kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`
cuser@amber:~$ curl $INGRESS_EXTERNAL_IP
 
cuser@amber:~$ kubectl delete deployment nginx
cuser@amber:~$ kubectl delete svc nginx
```

* You need to make sure that the source IP address (external-ip assigned by metallb) is preserved. To achieve this, set the value of the externalTrafficPolicy field of the ingress-controller Service spec to Local. At this point is ingress not installed, so please take a look on the ingress part.

```console
cuser@amber:~$ kubectl edit svc ingress-nginx-controller -n ingress-nginx
```

* change the default value for externalTrafficPolicy field 'Cluster' to Local:

```console
externalTrafficPolicy: Local
```

* optional: install with microk8s instead of helm(disadvantage is not separating namespaces):
```console
cuser@amber:~$ microk8s enable metallb:192.168.210.200-192.168.210.254
```
	
## Install Let's Encrypt 
* https://homekube.org/docs/cert-manager.html
* https://cert-manager.io/docs/configuration/acme/dns01/acme-dns/
* https://knowledge.digicert.com/solution/configure-cert-manager-and-digicert-acme-service-with-kubernetes
```console
cuser@amber:~$ sudo apt-get install jq -y
cuser@amber:~$ cd ~/
cuser@amber:~$ curl -s -X POST https://auth.acme-dns.io/register | jq . > acme-dns.json
cuser@amber:~$ vim ~/acme-dns.json
```
```console
{
  "username": "3e482c71-6eb7-4ac3-a761-82b0a97ec036",
  "password": "xP198CaYwgLMPH5c48on2F8-f1iZVf_dPJvMpnQ-",
  "fulldomain": "r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io",
  "subdomain": "r2120c32-20a2-3c7d-4f01-34fe4ttb3899",
  "allowfrom": []
}
```

* the domain we bought is <example.cloud>
* we take the value for the key "fulldomain" - for example: "r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io"
* we take the value and terminate it with a dot: r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io.
* we go to our hosting provider(for example exohosting.sk) and
* if existing A entry *.<example.cloud> then delete this entry and then we create a new cname entry there:
```console
name: _acme-challenge.<example.cloud>
value: r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io.
```

* (full output: _acme-challenge.<example.cloud>   IN CNAME	r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io.)
* (what we do not delete is <example.cloud>. type A targeting the IP {replace-this-with-your-public-ip})
* we check if it works:
```console
cuser@amber:~$ dig @8.8.8.8 _acme-challenge.<example.cloud>
```
* (output containing this should come: _acme-challenge.<example.cloud>. 3600 IN    CNAME   r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io.)

* (when we try dig _acme-challenge.<example.cloud> its ok, that we dont get the response, because normally the DNS 192.168.210.1 would be used):
```console
cuser@amber:~$ dig _acme-challenge.<example.cloud>
```

* create a new json file acme-dns-amber.json (using saved acme-dns.json) with this command(replace domain with yours):

```console
cuser@amber:~$ jq -n --arg domain "<example.cloud>" --argjson acme "$(cat ~/acme-dns.json)" '.+={($domain): $acme}' > ~/acme-dns-amber.json
```

* check if the file content is ok(check if the domain is correct => without <> parentheses):
```console
cuser@amber:~$ cat acme-dns-amber.json
{
  "<example.cloud>": {
    "username": "3e482c71-6eb7-4ac3-a761-82b0a97ec036",
    "password": "xP198CaYwgLMPH5c48on2F8-f1iZVf_dPJvMpnQ-",
    "fulldomain": "r2120c32-20a2-3c7d-4f01-34fe4ttb3899.auth.acme-dns.io",
    "subdomain": "r2120c32-20a2-3c7d-4f01-34fe4ttb3899",
    "allowfrom": []
  }
}
```

* (for multiple domains please read: https://cert-manager.io/docs/configuration/acme/dns01/acme-dns/) 
* create a secret in the cert-manager namespace(we have this because cert-manager was already installed):
```console
cuser@amber:~$ kubectl create secret generic acme-dns-amber -n cert-manager-<example-cloud> --from-file acme-dns-amber.json
```
* wait(few minutes) until cname is propagated: dig _acme-challenge.<example.cloud>
* (Requesting a certificate is accomplished by creating an instance of Certificate)
```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-<example-cloud>

---

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <example>-certificate-staging
  namespace: cert-manager-<example-cloud>
spec:
  dnsNames:
    - '<example.cloud>'
    - '*.<example.cloud>'
  secretName: <example>-tls-staging
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer

---

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: {replace-this-with-your-email}
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Store LetsEncrypt private identification keys here. Read more ...
      # https://github.com/openshift/release/pull/9372
      name: cert-issuer-account-key
    solvers:
      - dns01:
          # this is important for the verification self-check
          cnameStrategy: Follow
          acmeDNS:
            host: https://auth.acme-dns.io
            accountSecretRef:
              name: acme-dns-amber
              key: acme-dns-amber.json
        selector:
          dnsZones:
            - '<example.cloud>'
EOF
```

* (we need to wait 2 minutes..)
* we can check the clusterissuer with:
```console
cuser@amber:~$ kubectl describe clusterissuer letsencrypt-staging
```

* Once the ACME account is registered check the certificate request status:
```console
cuser@amber:~$ kubectl describe certificaterequest -n cert-manager-<example-cloud>
```

* To check the certificate status:
```console
cuser@amber:~$ kubectl describe certificate -n cert-manager-<example-cloud>
```

```console
cuser@amber:~$ kubectl get secrets -n cert-manager-<example-cloud> --watch
cuser@amber:~$ kubectl describe secret <example>-tls-staging-12345 -n cert-manager-<example-cloud>
```
->The important part here is that both tls.crt and tls.key must be present and not empty. This may take a while until the tls.crt is present and its size is > 0 !.

* if there are some problems popping out under kubectl describe clusterissuer letsencrypt-staging we can check the logs:
```console
cuser@amber:~$ kubectl get pods -n cert-manager
cuser@amber:~$ kubectl logs cert-manager-12345 -n cert-manager
```

* most of the time the problem is DNS, follow https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

* **NOW THE PROD VERSION**(replaced acme server endpoint and staging by prod):

* check if the cert-manager secret is there:
```console
cuser@amber:~$ kubectl describe secret acme-dns-amber -n cert-manager
```

```console
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-<example-cloud>

---

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <example>-certificate-prod
  namespace: cert-manager-<example-cloud>
spec:
  dnsNames:
    - '<example.cloud>'
    - '*.<example.cloud>'
  secretName: <example>-tls-prod
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

---

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: {replace-this-with-your-email}
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Store LetsEncrypt private identification keys here. Read more ...
      # https://github.com/openshift/release/pull/9372
      name: cert-issuer-account-key
    solvers:
      - dns01:
          # this is important for the verification self-check
          cnameStrategy: Follow
          acmeDNS:
            host: https://auth.acme-dns.io
            accountSecretRef:
              name: acme-dns-amber
              key: acme-dns-amber.json
        selector:
          dnsZones:
            - '<example.cloud>'
EOF
```

* (we need to wait 2 minutes..)
* we can check the clusterissuer with:
```console
cuser@amber:~$ kubectl describe clusterissuer letsencrypt-prod
cuser@amber:~$ kubectl describe secret <example>-tls-prod-12345 -n cert-manager-<example-cloud>
```
-> tls key and crt should have a size > 0

* for uninstall:
```console
cuser@amber:~$ kubectl delete secret acme-dns-amber -n cert-manager
cuser@amber:~$ kubectl delete certificate <example>-certificate-staging -n cert-manager-<example-cloud>
cuser@amber:~$ kubectl delete clusterIssuer letsencrypt-staging
cuser@amber:~$ kubectl delete certificate <example>-certificate-prod -n cert-manager-<example-cloud>
cuser@amber:~$ kubectl delete clusterIssuer letsencrypt-prod
cuser@amber:~$ kubectl get Issuers,ClusterIssuers,Certificates,CertificateRequests,Orders,Challenges --all-namespaces
cuser@amber:~$ helm uninstall --namespace cert-manager cert-manager
cuser@amber:~$ kubectl delete namespace cert-manager
```

## Install ingress
Internet -> LoadBalancer ->  Ingress -> service
```console
cuser@amber:~$ kubectl get ingress --all-namespaces
cuser@amber:~$ kubectl -n ingress-nginx get svc
```
* if there are any services, then clean: 
```console
cuser@amber:~$ kubectl delete svc nginx-helm-nginx-ingress -n ingress-nginx, kubectl delete svc ingress -n ingress-nginx
```
* if there is a namespace delete it:
```console
cuser@amber:~$ kubectl delete namespace ingress-nginx
```

```console
cuser@amber:~$ helm ls --all-namespaces
cuser@amber:~$ microk8s disable ingress
cuser@amber:~$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
cuser@amber:~$ helm repo update
cuser@amber:~$ kubectl create namespace ingress-nginx
```
* for next steps install reflector first(or make the changes after reflector is installed):
* add the ingress-nginx under secretTemplate reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces
```console
cuser@amber:~$ kubectl edit certificate <example>-certificate-staging -n cert-manager-<example-cloud>
cuser@amber:~$ kubectl edit certificate <example>-certificate-prod -n cert-manager-<example-cloud>
```
* add the ingress-nginx under secretTemplate reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces
```console
cuser@amber:~$ kubectl get secrets -n ingress-nginx
```

* make a file ingress-helm-values.yaml with the content: 
```console
cuser@amber:~$ cat ingress-helm-values.yaml
```

```
controller:
  metrics:
    enabled: true
    service:
      # important for detection of nginx datasource by prometheus
      annotations: {
        prometheus.io/scrape: "true" ,
      }
```

* install ingress
```console
cuser@amber:~$ helm install ingress-nginx ingress-nginx/ingress-nginx -f ingress-helm-values.yaml \
--version 4.12.0 \
--namespace ingress-nginx \
--set controller.extraArgs.default-ssl-certificate="ingress-nginx/<example>-tls-prod" \
--set controller.service.loadBalancerIP=192.168.210.200

cuser@amber:~$ kubectl get pods -n ingress-nginx
```	
-> if errors then check logs with(use your controller name): 
```console
cuser@amber:~$ kubectl logs ingress-nginx-controller-5864d666b8-mjkw6 -n ingress-nginx
```

* check if we have the "nginx" ingress class
```console
cuser@amber:~$ kubectl get ingressclass -n ingress-nginx
```

## Install rancher
* https://artifacthub.io/packages/helm/rancher-stable/rancher \
* https://cert-manager.io/docs/installation/helm/#option-2-install-crds-as-part-of-the-helm-release

```console
cuser@amber:~$ helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
cuser@amber:~$ helm repo update

cuser@amber:~$ helm install rancher rancher-stable/rancher \
--version 2.9.2 \
--namespace cattle-system \
--create-namespace \
--set global.cattle.psp.enabled=false \
--set hostname=rancher.<example.cloud> \
--set replicas=1 \
--set bootstrapPassword=Changeme123

cuser@amber:~$ kubectl get svc -n cattle-system

cuser@amber:~$ kubectl get ingress -n cattle-system
cuser@amber:~$ kubectl edit ingress rancher -n cattle-system
```

* change the spec to:
```console
  ingressClassName: nginx
  rules:
  - host: rancher.<example.cloud>
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: rancher
              port:
                number: 80
```

* change the secret to <example>-tls-prod
* check for running pods:
```console
cuser@amber:~$ kubectl get pods --namespace cattle-system
```
* check deploy:
```console
cuser@amber:~$ kubectl -n cattle-system rollout status deploy/rancher
```
## Install dashboard
```console
cuser@amber:~$ microk8s disable dashboard
cuser@amber:~$ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
cuser@amber:~$ helm repo update
cuser@amber:~$ helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
--namespace kubernetes-dashboard \
--create-namespace
```

```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ingressClassName: nginx
  rules:
  - host: dashboard.<example.cloud>
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kubernetes-dashboard
              port:
                number: 443
EOF
```

```console
cuser@amber:~$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
* open the browser, go to dashboard, chose radio button token,
* copy the token value and paste it into form then click sign in. You’ll be able to login with admin permission.
* navigate to https://dashboard.<example.cloud>/

## Install reflector
* firstly check if all namespaces are present

```console
cuser@amber:~$ helm repo add emberstack https://emberstack.github.io/helm-charts
cuser@amber:~$ helm repo update
cuser@amber:~$ helm install reflector emberstack/reflector \
--version 7.1.288 \
--namespace reflector \
--create-namespace
```
* as described here: https://cert-manager.io/docs/tutorials/syncing-secrets-across-namespaces/ add secretTemplate under spec:
* you can read more about reflector: https://github.com/emberstack/kubernetes-reflector/blob/main/README.md
```console
cuser@amber:~$ kubectl edit certificate <example>-certificate-staging -n cert-manager-<example-cloud>
cuser@amber:~$ kubectl edit certificate <example>-certificate-prod -n cert-manager-<example-cloud>
```

```console
  secretTemplate:
    annotations:
      reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
      reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "cattle-system,ingress-nginx,kube-system,longhorn-system,gitea,nextcloud,jupyterhub,lms,mqtt,node-red,home-assistant,influxdb,grafana"  # Control destination namespaces
      reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true" # Auto create reflection for matching namespaces
      reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "cattle-system,ingress-nginx,kube-system,longhorn-system,gitea,nextcloud,jupyterhub,lms,mqtt,node-red,home-assistant,influxdb,grafana" # Control auto-reflection namespaces
```

## Install longhorn
* we need a special path, same on every node - in this case /mnt/ssd:
```console
cuser@amber:~$ sudo lvcreate -n ssd -L 700G ubuntu-vg
cuser@amber:~$ sudo mkfs.ext4 /dev/ubuntu-vg/ssd
cuser@amber:~$ sudo mkdir /mnt/ssd
cuser@amber:~$ sudo blkid
cuser@amber:~$ sudo vim /etc/fstab
```

```console
cuser@vg:~$ sudo lvcreate -n ssd -L 700G ubuntu-vg
cuser@vg:~$ sudo mkfs.ext4 /dev/ubuntu-vg/ssd
cuser@vg:~$ sudo mkdir /mnt/ssd
cuser@vg:~$ sudo blkid
cuser@vg:~$ sudo vim /etc/fstab
```

* add entry to fstab:
```console
/dev/disk/by-id/dm-uuid-LVM-JLKtipf6uNcwEXfTT9ZMkwhylkFs77MCl0RQqJO6PVC41rJWn6JADDmxTWyBqqQJ /mnt/ssd ext4 defaults 0 1
```

* take a look at https://longhorn.io/kb/troubleshooting-volume-with-multipath/
* for every node:
```console
cuser@amber:~$ sudo apt install -y open-iscsi containerd nfs-common jq
cuser@amber:~$ sudo systemctl enable open-iscsi
```

```console
cuser@vg:~$ sudo apt install -y open-iscsi containerd nfs-common jq
cuser@vg:~$ sudo systemctl enable open-iscsi
```

* for master node:
```console
cuser@amber:~$ ls /var/snap/microk8s
```

-> the directory name with a number will differ, here it was 4055
```console
cuser@amber:~$ ls -la /var/snap/microk8s/4055/credentials/ | grep client.config
```

```console
-rw-rw---- 1 root microk8s 1870 Oct  7 14:15 client.config
```

```console
cuser@amber:~$ sudo chmod g+r /var/snap/microk8s/4055/credentials/client.config
cuser@amber:~$ sudo shutdown -r now

cuser@amber:~$ helm repo add longhorn https://charts.longhorn.io
cuser@amber:~$ helm repo update
```

* loadbalancer->ingress->svc
```console
cuser@amber:~$ apt-get install open-iscsi
cuser@amber:~$ sudo apt-get install nfs-common

cuser@amber:~$ kubectl delete --all pods --namespace=longhorn-system
```
	
* check for requirements: https://longhorn.io/docs/1.7.1/deploy/install/#installation-requirements

```console
cuser@amber:~$ curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.1/scripts/environment_check.sh | bash

cuser@amber:~$ kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.1/deploy/prerequisite/longhorn-iscsi-installation.yaml
cuser@amber:~$ kubectl -n longhorn-system get pod | grep longhorn-iscsi-installation
cuser@amber:~$ kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.1/deploy/prerequisite/longhorn-nfs-installation.yaml
cuser@amber:~$ kubectl -n longhorn-system get pod | grep longhorn-nfs-installation
cuser@amber:~$ kubectl -n longhorn-system logs longhorn-nfs-installation-hz647 -c nfs-installation
cuser@amber:~$ kubectl -n longhorn-system logs longhorn-iscsi-installation-tzq8x -c nfs-installation
```

* how to uninstall if needed: 
```console
cuser@amber:~$ kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.1/deploy/prerequisite/longhorn-iscsi-installation.yaml
cuser@amber:~$ kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.1/deploy/prerequisite/longhorn-nfs-installation.yaml
```

cuser@amber:~$ helm install longhorn longhorn/longhorn \
--namespace longhorn-system \
--create-namespace \
--set defaultSettings.defaultDataPath="/mnt/ssd" \
--set csi.kubeletRootDir=/var/snap/microk8s/common/var/lib/kubelet \
--set persistence.defaultClassReplicaCount=2 \
--version 1.7.1

cuser@amber:~$ kubectl -n longhorn-system get pod
```
* longhorn ui is default unprotected, and we want auth:

```console
cuser@amber:~$ cd /opt/longhorn
cuser@amber:~$ USER=cuser; PASSWORD={replace-this-with-your-password}; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
cuser@amber:~$ sudo chown cuser:cuser auth
cuser@amber:~$ kubectl -n longhorn-system create secret generic basic-auth --from-file=/opt/longhorn/auth
cuser@amber:~$ kubectl -n longhorn-system get secret basic-auth -o yaml
```

```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: longhorn-system
  name: longhorn
  annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # prevent the controller from redirecting (308) to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn.<example.cloud>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
  tls:
  - hosts:
    - longhorn.<example.cloud>
    secretName: <example>-tls-prod
EOF
```

```console
cuser@amber:~$ kubectl -n longhorn-system get ingress
```

* create a new StorageClass

```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-fast
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: "Delete"
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  diskSelector: "ssd"
  nodeSelector: "ssd"
EOF
```

```console
cuser@amber:~$ kubectl -n longhorn-system get ingress
```
-> from the output we know the hostname: longhorn.<example.cloud>

* in the UI change the storage class name to longhorn-fast
* and under Node->Operation->Edit node and discs->add tag ssd for both disk and node

```console
cuser@amber:~$ kubectl patch storageclass microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
cuser@amber:~$ kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
cuser@amber:~$ kubectl patch storageclass longhorn-fast -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
cuser@amber:~$ kubectl get storageclass
```

* howto degragated: https://github.com/longhorn/longhorn/issues/2067
* when I had problems it was dns: https://longhorn.io/kb/troubleshooting-dns-resolution-failed/
* how to debug dns: https://help.hcl-software.com/connections/v6/admin/install/cp_prereq_kubernetes_dns.html

## Install gitea 
* Optional: (this example is working with https, ingress is not able to forward ssh ports, if needed then a new LoadBalancer is required - on port 22 with annotation: metallb.universe.tf/allow-shared-ip: "{{ ndo_context }}")

```console
cuser@amber:~$ helm repo add gitea-charts https://dl.gitea.io/charts/

cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  namespace: gitea
  name: gitea-admin-secret
type: Opaque
stringData:
  username: {replace-this-with-your-admin-username-for-gitea}
  password: {replace-this-with-your-admin-password-for-gitea}
EOF
```
* create values-gitea.yml:
```console
cuser@amber:~$ cat values-gitea.yml
gitea:
  clusterDomain: gitea.<example.cloud>
  admin:
    existingSecret: gitea-admin-secret

persistence:
  storageClass: longhorn-fast

postgresql:
  global:
    postgresql:
      postgresqlPassword: {replace-this-with-your-password-for-gitea-db}

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: gitea.<example.cloud>
      paths:
        - path: /
          pathType: Prefix
  tls:
   - secretName: <example>-tls-prod
     hosts:
       - gitea.<example.cloud>

cuser@amber:~$ helm install --version 8.3.0 --namespace gitea gitea gitea-charts/gitea --values values-gitea.yml
cuser@amber:~$ kubectl get pods -n gitea
```

## Install nextcloud (https://github.com/nextcloud/helm)
```console
cuser@amber:~$ mkdir /nextcloud

cuser@amber:~$ kubectl get nodes --show-labels
```
* check if storagetype=fileserver is on the node where the BIG STORAGE DISK exists

```console
cuser@amber:~$ helm repo add bitnami https://charts.bitnami.com/bitnami
cuser@amber:~$ helm repo update

cuser@amber:~$ helm repo add nextcloud https://nextcloud.github.io/helm/
cuser@amber:~$ helm repo update
```

* values for nextcloud chart: https://artifacthub.io/packages/helm/nextcloud/nextcloud
* (values for postgres chart from bitnami - which are not important here - can be found also under: https://artifacthub.io/packages/helm/bitnami/postgresql)
* !!! before mounting !!! check that the owner of /nextcloud is root, if not:

```console
cuser@amber:~$ sudo chown -R root:root /nextcloud
cuser@amber:~$ sudo chmod 755 /nextcloud
```
-> the reason is, that we want to mount a crypted hdd cointaining data to /nextcloud with this script:

```console
cuser@amber:~$ touch /media/mount-fileserver-data.sh
cuser@amber:~$ sudo chmod +x /media/mount-fileserver-data.sh
cuser@amber:~$ cat mount-fileserver-data.sh

# mount-fileserver-data.sh
#!/bin/bash
if [[ $(id -u) -ne 0 ]] ; then echo "Please run as root with: sudo ./mount-fileserver-data.sh" ; exit 1 ; fi
# Read Password
echo -n Please enter the cryptsetup password for sda:
read -s password
echo
# Run Command
echo $password | cryptsetup luksOpen /dev/disk/by-uuid/0741ff3b-9f83-4fdb-8c5d-3218fb0fabc4 crypted_sda
lvscan
mount /dev/vga01/lv00_main /nextcloud -o user=www-data,rw
```
* after mounting change the owner of all files on the data hdd(this enables to edit/delete current data from nextcloud 

```console
cuser@amber:~$ sudo chown -R www-data:www-data /nextcloud
```
* The lowercase s bit on the group permission means that the directory is setguid. Any directory created in that directory will belong to the same group as its parents directory instead of the default group of the user that created it
```console
cuser@amber:~$ sudo chmod -R u=rwx,g=rx+s,o=rx /nextcloud

cuser@amber:~$ sudo usermod -a -G www-data {replace-this-with-your-system-username}
cuser@amber:~$ groups {replace-this-with-your-system-username}
```

* create values-nextcloud.yml:

```console
cuser@amber:~$ cat values-nextcloud.yml
image:
  flavor: fpm
ingress:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-headers: "X-Forwarded-For"
	nginx.ingress.kubernetes.io/proxy-body-size: 20240m
  enabled: true
  className: nginx
  path: /
  pathType: Prefix
  tls:
   - secretName: <example>-tls-prod
     hosts:
       - nextcloud.<example.cloud>
nextcloud:
  host: nextcloud.<example.cloud>
  username: {replace-this-with-your-username-for-nextcloud}
  password: {replace-this-with-your-password-for-nextcloud}
  extraVolumes:
    - name: nextcloud-external-volume
      hostPath:
        path: /nextcloud
        type: Directory
  extraVolumeMounts:
    - name: nextcloud-external-volume
      mountPath: /external-data
nginx:
  enabled: true
internalDatabase:
  enabled: false
mariadb:
  enabled: false
postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        password: {replace-this-with-your-username-for-nextcloud-db} 
    storageClass: longhorn-fast
  primary:
    persistence:
      enabled: true
      size: 20Gi
      accessMode: ReadWriteMany
persistence:
  enabled: true
  storageClass: longhorn-fast
  size: 300Gi
phpClientHttpsFix:
  enabled: true
  protocol: https
```

```console
cuser@amber:~$ helm install nextcloud nextcloud/nextcloud \
--version v3.5.12  \
--namespace nextcloud \
--values values-nextcloud.yml \
--set nodeSelector.storagetype=fileserver

cuser@amber:~$ watch kubectl get pods -n nextcloud

cuser@amber:~$ kubectl get pods -n nextcloud
```

* optional-check if www-data users have the same id:
* open container shell for the pod(in this example we copied nextcloud-599dcdbc67-gbbgg)
```console
cuser@amber:~$ kubectl -n nextcloud exec --stdin --tty nextcloud-599dcdbc67-gbbgg -- /bin/bash
- and inside the container run:
$ id -u www-data
$ id -g www-data

or

cuser@amber:~$ kubectl -n nextcloud exec -it nextcloud-599dcdbc67-gbbgg -- /bin/bash -c "id -u www-data"
cuser@amber:~$ kubectl -n nextcloud exec -it nextcloud-599dcdbc67-gbbgg -- /bin/bash -c "id -g www-data"

- check if user and group www-data with corresponding id exists
$ id -u www-data
$ id -g www-data

- if does not exit add:
$ sudo groupadd -g 33 www-data
$ sudo useradd www-data -u 33 -g 33 -m -s /bin/false
```

* go to nextcloud.<example.cloud> and under Apps add External storage
* go to User and Add group fileserver
* go to User->Active users, click edit and add group fileserver, then go to Settings->Administration->External storage and add nextcloud as Local with Configuration /external-data and for fileserver group

```console
cuser@amber:~$ kubectl scale deployment nextcloud --replicas=0 -n nextcloud
cuser@amber:~$ kubectl scale deployment nextcloud --replicas=1 -n nextcloud
```	  
-> remounting after restart to see data
	  
## Install rsync
* on the server which will send backups
```console
cuser@amber:~$ sudo apt-get install rsync
```

* on backup server:
```console
cuser@vg:~$ sudo mkdir /nextcloud
cuser@vg:~$ sudo chown -R root:root /nextcloud
cuser@vg:~$ sudo chmod 755 /nextcloud
cuser@vg:~$ sudo usermod -a -G www-data {replace-this-with-your-system-username}
```

* then call the mount script unter /media on backup server

one time run - after successful mount!:
```console
cuser@vg:~$ ls -la /nextcloud/
cuser@vg:~$ sudo chown -R www-data:www-data /nextcloud
cuser@vg:~$ sudo chmod -R u=rwx,g=rx+s,o=rx /nextcloud
```

```console
cuser@amber:~$ rsync -arvz -e "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" --progress --delete /nextcloud/ {replace-this-with-your-system-username}@vg.<example.cloud>:/nextcloud/
```

* to create a patch to usb
```console
cuser@amber:~$ cd /media/usb
cuser@amber:~$ sudo rsync -arv --only-write-batch=patch /nextcloud/ {replace-this-with-your-system-username}@vg.<example.cloud>:/nextcloud/
```

* to apply it
```console
cuser@vg:~$ rsync -arv --read-batch=patch /nextcloud/

or use an auto-generated script(do not forget to check if target location is correctly mounted):

cuser@vg:~$ sudo ./patch.sh
```

## Install smartmontools, smartd

* on all nodes

```console
cuser@amber:~$ sudo apt-get install smartmontools
cuser@amber:~$ sudo apt-get update && sudo apt-get install msmtp

cuser@amber:~$ sudo touch /etc/msmtprc
cuser@amber:~$ sudo chmod 644 /etc/msmtprc
```

```console
cuser@amber:~$ sudo vim /etc/msmtprc

defaults
auth on
tls  on
tls_starttls off
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /var/log/msmtp.log

account <example.cloud>
host    {replace-this-with-your-smtp-host}
port    {replace-this-with-your-smtp-port}
from    info@<example.cloud>
user    info@<example.cloud>
password {replace-this-with-your-smtp-password}

account default: <example.cloud>
```

* install mailutils:

```console
cuser@amber:~$ sudo apt install mailutils
```

* install sendmail:
```console
cuser@amber:~$ sudo apt install msmtp-mta
```

```console
cuser@amber:~$ sudo chmod u=rw,g=r,o=r /etc/mail.rc
cuser@amber:~$ sudo vim /etc/mail.rc
```
and add this line:
```console
set sendmail="/usr/bin/msmtp -t"
```

* test 1:
```console
cuser@amber:~$ echo "Hello World" | msmtp -d info@<example.cloud>
```

* test 2:
```console
cuser@amber:~$ echo -e "Subject: This is a Test" | sendmail info@<example.cloud> -F servers-hostname
```

* test 3:
```console
cuser@amber:~$ echo "Testing msmtp from ${HOSTNAME} with mail command" | mail -s "hi there" info@<example.cloud>
```

* Install smartmontools

```console
cuser@amber:~$ sudo vim /etc/default/smartmontools
```
add this line:
```console
start_smartd=yes
```

* test it: comment out the DEVICESCAN line in /etc/smartd.conf and add:
```console
/dev/sda -a -m info@<example.cloud> -M test
```
* then test it:
```console
cuser@amber:~$ sudo systemctl restart smartd
```
* if works then change it to(min 35, max 55Celsia):
```console
DEVICESCAN -H -l error -l selftest -f -o on -s (O/../../5/11|L/../../5/13|C/../../5/15) -W 4,35,55 -d removable -m info@<example.cloud> -n standby -M exec /usr/share/smartmontools/smartd-runner
```

## Install jupyterhub
* https://artifacthub.io/packages/helm/jupyterhub/jupyterhub
```console
cuser@amber:~$ helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
cuser@amber:~$ helm repo update
cuser@amber:~$ vim values-jupyterhub2.yml
```

```console
hub:
  config:
    Authenticator:
      admin_users:
        - {replace-this-with-your-jupyter-user}
    DummyAuthenticator:
      password: {replace-this-with-your-jupyter-password}
    JupyterHub:
      authenticator_class: dummy
  db:
    pvc:
      storageClassName: longhorn-fast
    password: {replace-this-with-your-jupyter-db-password}
proxy:
  service:
    type: ClusterIP
singleuser:
  storage:
    dynamic:
      storageClass: longhorn-fast
    capacity: 2Gi
  extraEnv:
    GRANT_SUDO: "yes"
  uid: 0
  fsGid: 0
  cmd: ["jupyterhub-singleuser", "--allow-root"]
ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
   - jupyterhub.<example.cloud>
  tls:
   - secretName: <example>-tls-prod
     hosts:
       - jupyterhub.<example.cloud>
```
* install jupyterhub:
```console
cuser@amber:~$ helm install --version 2.0.0 --namespace jupyterhub jupyterhub jupyterhub/jupyterhub --values values-jupyterhub2.yml
```
* wait for all pods and then navigate to jupyterhub.<example.cloud> and login using {replace-this-with-your-jupyter-user} and {replace-this-with-your-jupyter-password}
* install pandoc and tex inside the user pod(jupyter-{replace-this-with-your-jupyter-user})
```console
cuser@amber:~$ kubectl -n jupyterhub exec --stdin --tty jupyter-{replace-this-with-your-jupyter-user} -- /bin/bash

$ apt-get update
$ apt-get install pandoc texlive-xetex
```

## Install logitech media server
```console
cuser@amber:~$ helm repo add cronce https://charts.cronce.io/
cuser@amber:~$ helm install lms cronce/logitech-media-server \
--version 0.1.2 \
--namespace lms \
--create-namespace \
--set persistence.config.storageClass=longhorn-fast \
--set settings.timezone="Europe/Bratislava" \
--set nodeSelector.storagetype=fileserver
```
```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: lms
  name: lms
spec:
  ingressClassName: nginx
  rules:
  - host: lms.<example.cloud>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: lms-logitech-media-server
            port:
              number: 9000
  tls:
  - hosts:
    - lms.<example.cloud>
    secretName: <example>-tls-prod
EOF
```

```console
cuser@amber:~$ kubectl edit statefulsets.apps/lms-logitech-media-server -n lms
	  
      volumes:
      - name: music
        hostPath:
          path: /nextcloud/{replace-this-with-your-path-to-music-under-nextcloud}
          type: Directory
```

```console
cuser@amber:~$ kubectl get all -n lms
```

## Install rabbitmq
* https://artifacthub.io/packages/helm/bitnami/rabbitmq
* https://www.rabbitmq.com/community-plugins.html

```console
$ cat values-mqtt.yaml
auth:
  username: {replace-this-with-your-rabbitmq-user}
  password: {replace-this-with-your-rabbitmq-password}
extraConfiguration: |-
    mqtt.allow_anonymous = false
ingress:
  enabled: true
  hostname: rabbitmq.<example.cloud>
  tls: true
  ingressClassName: nginx
  extraTls:
    - secretName: <example>-tls-prod
      hosts:
        - rabbitmq.<example.cloud>
persistence:
  enabled: true
  storageClass: longhorn-fast
extraPlugins: "rabbitmq_auth_backend_ldap,rabbitmq_mqtt,rabbitmq_web_mqtt"
service:
  extraPorts:
    - name: mqtt-tcp
      port: 1883
      targetPort: 1883
    - name: mqtt-web
      port: 15675
      targetPort: 15675
```
* install rabbitmq
```console
cuser@amber:~$ helm install rabbitmq bitnami/rabbitmq --version 11.14.4  --namespace mqtt --values values-mqtt.yaml
```
* login and load the definition:
```console
{
  "users": [
	  {
		  "name": "amqp",
		  "password": "{replace-this-with-your-rabbitmq-amqp-password}",
		  "tags": "amqp-client"
	  },
	  {
		  "name": "{replace-this-with-your-rabbitmq-mqtt-user}",
		  "password": "{replace-this-with-your-rabbitmq-mqtt-password}",
		  "tags": "mqtt-client"
	  }
  ],
  "vhosts": [
	  {
		  "name": "amqp"
	  },
	  {
		  "name": "mqtt"
	  }
  ],
  "permissions": [
	  {
		  "user": "amqp",
		  "vhost": "amqp",
		  "configure": ".*",
		  "write": ".*",
		  "read": ".*"
	  },
	  {
		  "user": "{replace-this-with-your-rabbitmq-mqtt-user}",
		  "vhost": "mqtt",
		  "configure": ".*",
		  "write": ".*",
		  "read": ".*"
	  },
	  {
		  "user": "{replace-this-with-your-rabbitmq-mqtt-user}",
		  "vhost": "/",
		  "configure": ".*",
		  "write": ".*",
		  "read": ".*"
	  }
  ]
}
```
* from the graphical interface under rabbitmq.<example.cloud> choose a json with the content above and click on upload button to merge the new definitions
```console
cuser@amber:~$ kubectl get pods --show-labels -n mqtt
cuser@amber:~$ kubectl get pod --selector="app.kubernetes.io/name=rabbitmq" -n mqtt
```
```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-lb
  namespace: mqtt
spec:
  selector:
    app.kubernetes.io/name: rabbitmq
  type: LoadBalancer
  ports:
    - name: rabbitmq
      protocol: TCP
      port: 1883
      targetPort: 1883
EOF
```
* get the external ip of the rabbitmq-lb:
```console
cuser@amber:~$ kubectl get svc -n mqtt
```
* install mosquitto-2.0.15-install-windows-x64.exe from https://mosquitto.org/download/
* open cmd and cd C:\Program Files\mosquitto
```console
C:\Program Files\mosquitto\mosquitto_pub.exe -h 192.168.210.201 -t topic -m "Hello" -u {replace-this-with-your-rabbitmq-user} -P {replace-this-with-your-rabbitmq-password} -d
C:\Program Files\mosquitto\mosquitto_sub.exe -h 192.168.210.201 -t topic -u {replace-this-with-your-rabbitmq-user} -P {replace-this-with-your-rabbitmq-password}

curl -i -u {replace-this-with-your-rabbitmq-mqtt-user}:{replace-this-with-your-rabbitmq-mqtt-password} http://192.168.210.10:15672/api/vhosts
```
## Install influxdb
* https://artifacthub.io/packages/helm/bitnami/influxdb
```console
cuser@amber:~$ helm repo add bitnami https://charts.bitnami.com/bitnami
cuser@amber:~$ helm repo update
cuser@amber:~$ kubectl create namespace node-red
```
* do not forget to sync the ingress secret with reflector
```console
cuser@amber:~$ cat values-influxdb.yaml
```
```console
clusterDomain: <example.cloud>
global:
  storageClass: longhorn-fast
ingress:
  enabled: true
  hostname: influxdb.<example.cloud>
  ingressClassName: nginx
  extraTls:
    - secretName: <example>-tls-prod
      hosts:
        - influxdb.<example.cloud>
persistence:
  storageClass: longhorn-fast
```
* install influxdb:
```console
cuser@amber:~$ helm install influxdb oci://registry-1.docker.io/bitnamicharts/influxdb --version 5.5.2 --namespace influxdb --create-namespace -f values-influxdb.yaml

cuser@amber:~$ kubectl get pods -n influxdb
cuser@amber:~$ kubectl -n influxdb exec -it influxdb-5b65cc7f5b-xs6cl -- cat /bitnami/influxdb/configs
```
* (get the default.token: "oFreOiXBQTtwNYIhD0Zp")
```console
cuser@amber:~$ kubectl -n influxdb exec -it influxdb-5b65cc7f5b-xs6cl -- influx user password -n admin -t oFreOiXBQTtwNYIhD0Zp
```
* (write the new pass)
```console
cuser@amber:~$ kubectl get pods --show-labels -n influxdb
cuser@amber:~$ kubectl get pod --selector="app.kubernetes.io/name=influxdb" -n influxdb
```

```console
cuser@amber:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: influxdb-lb
  namespace: influxdb
spec:
  selector:
    app.kubernetes.io/name: influxdb
  type: LoadBalancer
  ports:
    - name: influxdb
      protocol: TCP
      port: 8086
      targetPort: 8086
EOF
```
```console
cuser@amber:~$ kubectl get svc -n influxdb
```

## Install grafana
* https://artifacthub.io/packages/helm/grafana/grafana
```console
cuser@amber:~$ helm repo add grafana https://grafana.github.io/helm-charts
cuser@amber:~$ helm repo update
cuser@amber:~$ helm install my-release grafana/grafana
cuser@amber:~$ kubectl create namespace grafana
```
* do not forget to sync the ingress secret with reflector

```console
cuser@amber:~$ cat values-grafana.yaml
ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
    - grafana.<example.cloud>
  tls:
   - secretName: <example>-tls-prod
     hosts:
       - grafana.<example.cloud>
persistence:
  enabled: true
  storageClassName: longhorn-fast
```
* install grafana
```console
cuser@amber:~$ helm install grafana grafana/grafana --version 6.56.2 --namespace grafana --create-namespace -f values-grafana.yaml
```
* user is admin and pass:
```console
cuser@amber:~$ kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
* add source influxdb with Query Language: Flux, URL: http://192.168.210.202:8086

## Install nodered
* https://artifacthub.io/packages/helm/node-red/node-red
```console
cuser@amber:~$ helm repo add node-red https://schwarzit.github.io/node-red-chart/
cuser@amber:~$ helm repo update
cuser@amber:~$ kubectl create namespace node-red
```
* do not forget to sync the ingress secret with reflector
```console
cuser@amber:~$ cat values-node-red.yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: node-red.<example.cloud>
      paths:
        - path: /
          pathType: Prefix
  tls:
   - secretName: <example>-tls-prod
     hosts:
       - node-red.<example.cloud>
	 certificate:
	   enabled: false
```
* install node-red
```console
cuser@amber:~$ helm install node-red node-red/node-red --version 0.23.1 --namespace node-red --create-namespace -f values-node-red.yaml
```

* Navigate to right menu -> manage palette -> Install -> node-red-contrib-influxdb
* This configuration is an example:
```console
add inject,
msq.topic=sewage
msg.payload={} {"name":"clamp-1", "current":100.4, "power":300.7}
inject once after 5 seconds then interval every 10s

msg.payload = {
    name: msg.payload.name,
    current: msg.payload.current,
    power: msg.payload.power
}
return msg;

switch msg.payload.current >= 2 -> 1 otherwise 2
change1 on set payload.status to number 1
change2 on set payload.status to number 0
both to influxdb out configured with http://192.168.210.202:8086 version v2 and new generated token for sewage in influxdb read and write, bucket: sewage, measurement: stations, time precision seconds, organisation: primary
```
  
## Install home-assistant
* https://artifacthub.io/packages/helm/geek-cookbook/home-assistant
* https://github.com/k8s-at-home/library-charts/blob/main/charts/stable/common/values.yaml

```console
cuser@amber:~$ helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
cuser@amber:~$ helm repo update
cuser@amber:~$ kubectl create namespace home-assistant
```
* do not forget to sync the ingress secret with reflector
```console
cuser@amber:~$ cat values-home-assistant.yaml
env:
  TZ: Europe/Bratislava
  
ingress:
  main:
    enabled: true
    annotations:
      nginx.org/websocket-services: home-assistant
    ingressClassName: nginx
    hosts:
      - host: home-assistant.<example.cloud>
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: <example>-tls-prod
        hosts:
          - home-assistant.<example.cloud>

persistence:
  config:
    enabled: true
    storageClass: longhorn-fast
  
postgresql:
  enabled: true
  postgresqlUsername: {replace-this-with-your-username-for-home-assistant-db}
  postgresqlPassword: {replace-this-with-your-password-for-home-assistant-db}
  persistence:
    primary:
      enabled: true
      storageClass: "longhorn-fast"
```
```console
cuser@amber:~$ helm install home-assistant geek-cookbook/home-assistant --namespace home-assistant --create-namespace -f values-home-assistant.yaml
cuser@amber:~$ kubectl get pods -n home-assistant
cuser@amber:~$ kubectl -n home-assistant exec -it home-assistant-74f688c4f9-6szdh -- ifconfig
```
* (inet addr: 10.1.196.26)
```console
cuser@amber:~$ kubectl -n home-assistant exec -it home-assistant-74f688c4f9-6szdh -- vi configuration.yaml
```
- after default add(use inet addr from above):
```console
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
    - 10.1.196.0/24
```
* restart:
```console
cuser@amber:~$ kubectl delete pod home-assistant-74f688c4f9-6szdh -n home-assistant
```
## Install docker
* https://docs.docker.com/engine/install/ubuntu/

```console
cuser@amber:~$ sudo apt-get update -y
cuser@amber:~$ sudo apt upgrade
cuser@amber:~$ sudo apt install docker.io
cuser@amber:~$ sudo systemctl enable docker
cuser@amber:~$ sudo systemctl start docker
```
* verify that docker is running:
```console
cuser@amber:~$ sudo systemctl status docker

cuser@amber:~$ sudo apt-get install docker-compose
cuser@amber:~$ sudo chmod +x /usr/local/bin/docker-compose
```

## Update whole system(for all nodes)
```console
cuser@amber:~$ sudo apt update && sudo apt upgrade -y
```

```console
cuser@vg:~$ sudo apt update && sudo apt upgrade -y
```

* think about every helm-chart compatibility!

* to list all channels
```console
cuser@amber:~$ snap info microk8s
cuser@vg:~$ snap info microk8s
```
* to upgrade an multi-node cluster:
```console
cuser@amber:~$ microk8s kubectl drain amber.<example.cloud>  --ignore-daemonsets
cuser@amber:~$ microk8s kubectl drain vg.<example.cloud>  --ignore-daemonsets
cuser@amber:~$ microk8s kubectl get po -A -o wide
cuser@amber:~$ sudo snap refresh microk8s --channel=1.32/stable
cuser@vg:~$ sudo snap refresh microk8s --channel=1.32/stable
cuser@amber:~$ microk8s.kubectl get no
cuser@amber:~$ microk8s kubectl uncordon vg.<example.cloud>
cuser@amber:~$ microk8s kubectl uncordon amber.<example.cloud>
cuser@amber:~$ microk8s kubectl get po -A -o wide
```
* to revert a failed upgrade attempt:
```console
cuser@amber:~$ sudo snap revert microk8s
cuser@vg:~$ sudo snap revert microk8s
```

* as helm was installed with snap it will update itself with the channel refresh, so it should be up to date:
```console
cuser@amber:~$ helm version
```

* to get all currenty used versions for installed charts:
```console
cuser@amber:~$ helm list --all-namespaces
```

```console
cuser@amber:~$ helm repo update
```

* to list the latest possible version of metallb:
```console
cuser@amber:~$ helm search repo metallb/metallb
```

* to get all info including min version of kubernetes for metallb:
```console
cuser@amber:~$ helm show chart metallb/metallb
```

* helm upgrade charts to latest https://helm.sh/docs/helm/helm_upgrade/ 
```console
cuser@amber:~$ helm upgrade metallb metallb/metallb \
--namespace metallb-system \
--reuse-values

cuser@amber:~$ helm upgrade cert-manager jetstack/cert-manager \
--namespace cert-manager \
--reuse-values

cuser@amber:~$ helm list --all-namespaces
```