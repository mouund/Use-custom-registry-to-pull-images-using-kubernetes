# Setup a private OCI registry and use it in K8S over HTTPs


## Setup the registry
* We will use this doc to deploy the registry, you can refer direclt to it if needed
https://distribution.github.io/distribution/about/deploying/

### Generate certificates for TLS

* Use this doc to generate the CA, then create your server.key and server.crt https://github.com/mouund/PKI-infrastructure-for-private-infra
```sh
$ mkdir -p certs
```
place them inside of the `certs` folder
Using docker, we will host the registry, first generate htpasswd
```sh
$ mkdir auth
$ docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpassword > auth/htpasswd
```
The filetree should be 
```sh
$ tree 
auth
└── htpasswd
certs
├── server.crt
└── server.key
```
Now let's run the docker container and map port to 5000 on host
```sh
docker run -d -p 5000:5000 --restart=always --name registry -v "$(pwd)"/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 -v "$(pwd)"/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt -e REGISTRY_HTTP_TLS_KEY=/certs/server.key registry
```
Check that the container is running
```sh
$ docker ps 
CONTAINER ID   IMAGE      COMMAND                  CREATED        STATUS        PORTS                                       NAMES
d940cb5fe189   registry   "/entrypoint.sh /etc…"   12 hours ago   Up 12 hours   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   registry
```
### Setup SSL and name resolution
First, we need to trust the CA on ALL the machines that will be interacting with the registry over HTTPs (K8s nodes, local machine to push)
On each machine, 
```sh
apt-get install -y ca-certificates
cp ca.crt /usr/local/share/ca-certificates
update-ca-certificates
```
Where ca.crt is the ca certificate
> [!WARNING]  
> YOu need to restart the container engine, if not done, your CA won't be trusted

After that restart the container engine
For docker
```sh
systemctl restart docker
```
For k8s nodes it is generally containerd
```sh
systemctl restart containerd
```
```sh
systemctl restart kubelet
```
Finally on all machines, add this line to you host file
<IP of the machine hosting the container> <DNS name you used while generating the server.crt (in the server.config.DNS.1)>
eg.
```
echo 192.168.1.15 oci-image-home | sudo tee -a /etc/hosts
```
> [!WARNING]  
> The DNS name has to match the one setup in certificate Subject ALternative Names

YOu can check them by running 
```sh
$ openssl x509 -noout -ext subjectAltName -in certs/server.crt 
X509v3 Subject Alternative Name: 
    DNS:oci-image-home, IP Address:192.168.1.15
```
Here you see that the name I'll use for resolution is in the certificates ! Very important

## Use the registry locally to push image
Use this Dockerfile to build a dummy image
```sh
cat << EOF > Dockerfile 
FROM alpine:3.14
RUN apk add --no-cache curl
ENTRYPOINT ["curl", "8.8.8.8"] 
EOF
```
```sh
docker build . -t oci-image-home:5000/test-image:1
```
Now you can login 

```sh
docker login oci-image-home:5000
```
Username name and password are the one we've put earlier in the setup

FInally you can push your dummy image
```sh
docker push oci-image-home:5000/test-image:1
```
It should be ok !

## Use the registry in K8s
First create an Image pull secret
```sh
$ kubectl create secret docker-registry oci-image-home --docker-server=oci-image-home:5000 --docker-username=testuser --docker-password=testpassword
secret/oci-image-home created
```
```sh
$ kubectl get secret
NAME             TYPE                             DATA   AGE
oci-image-home   kubernetes.io/dockerconfigjson   1      4s
moun@mounub~/pki-infra-homelab$ 
```
Now, simply use that secret as Image pull secret in a pod defintion
```sh
cat << EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: oci-image-home:5000/test-image:1
  imagePullSecrets:
  - name: oci-image-home
EOF
```
Finally apply the manifest 
```sh
kubectl apply -f pod.yaml
```
THe image will be pulled and the pod will be Running !

Voilà ! :D
Reference: 
* https://distribution.github.io/distribution/about/deploying/
* https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/