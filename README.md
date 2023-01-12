## Procedure
**Step 1:** Generate CA, certificates and keys  
```bash
# Generate CA crt and key
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=TMA Inc./CN=tmanet.com' -keyout tmanet.com.key -out tmanet.com.crt
# Generate Istio Gateway TLS crt and key
openssl req -out gw.tmanet.com.csr -newkey rsa:2048 -nodes -keyout gw.tmanet.com.key -subj "/CN=*.tmanet.com/O=DC"
openssl x509 -req -days 365 -CA tmanet.com.crt -CAkey tmanet.com.key -set_serial 0 -in gw.tmanet.com.csr -out gw.tmanet.com.crt
# Generate nginx server TLS crt and key
openssl req -out nginx.tmanet.com.csr -newkey rsa:2048 -nodes -keyout nginx.tmanet.com.key -subj "/CN=nginx.apps.svc.cluster.local/O=DC"
# openssl x509 -req -days 365 -extfile <(printf "subjectAltName=DNS:nginx.tmanet.com") -CA tmanet.com.crt -CAkey tmanet.com.key -set_serial 1 -in nginx.tmanet.com.csr -out nginx.tmanet.com.crt # Add SAN to certificate
openssl x509 -req -days 365 -CA tmanet.com.crt -CAkey tmanet.com.key -set_serial 1 -in nginx.tmanet.com.csr -out nginx.tmanet.com.crt
# Generate crt and key used by Istio Gateway to connect nginx with mTLS
# openssl req -out client.gateway.csr -newkey rsa:2048 -nodes -keyout client.gateway.key -subj "/CN=istio-ingressgateway-*/O=DC"
# openssl x509 -req -days 365 -CA tmanet.com.crt -CAkey tmanet.com.key -set_serial 2 -in client.gateway.csr -out client.gateway.crt
```
**Step 2:** Start  minikube
```bash
minikube start --kubernetes-version=v1.22.11
```
**Step 3:** Install Istioctl
```bash
curl -LO https://github.com/istio/istio/releases/download/1.13.8/istioctl-1.13.8-linux-amd64.tar.gz
tar -xf istioctl-1.13.8-linux-amd64.tar.gz
sudo install istioctl /usr/local/bin/
``` 
**Step 4:** Install Istio
```bash
curl -LO https://raw.githubusercontent.com/nathluu/istio-fleetman/master/deployment/azure/single-cluster/2-setup-k8s.sh
bash 2-setup-k8s.sh
```
**Step 5:** Create apps namespace with istio auto injection  
```bash
kubectl create namespace apps
kubectl label namespace apps istio-injection=enabled
```
**Step 6:** Create a secret for the ingress gateway  
```bash
kubectl create -n istio-system secret generic gw-credential --from-file=tls.key=gw.tmanet.com.key \
--from-file=tls.crt=gw.tmanet.com.crt --from-file=ca.crt=tmanet.com.crt
# kubectl create -n istio-system secret tls gw-credential --cert=gw.tmanet.com.crt --key=gw.tmanet.com.key --cacert=tmanet.com.crt # Another way to create k8s secret for TLS
# kubectl create -n istio-system secret generic client-credential --from-file=tls.key=client.gateway.key --from-file=tls.crt=client.gateway.crt --from-file=ca.crt=tmanet.com.crt # Use this command for mTLS
```
**Step 7:** Create configmap and secret for nginx server  
```bash
kubectl create -n apps configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
# kubectl create -n apps configmap nginx-configmap --from-file=nginx.conf=./nginx-mutual-tls.conf # mTLS enabled
kubectl create -n apps secret generic nginx-server-certs --from-file=tls.key=nginx.tmanet.com.key \
--from-file=tls.crt=nginx.tmanet.com.crt --from-file=ca.crt=tmanet.com.crt
```
**Step 8:** Create nginx deployment, service, gateway and virtualService  
```bash
kubectl apply -n apps -f apps.yaml
kubectl apply -n apps -f routes.yaml
kubectl apply -n apps -f destination.yaml
# kubectl apply -f destination-mutual-tls.yaml # Use this command for mTLS
```
**Step 9:** Open another terminal and run minikube tunnel
```bash
sudo minikube tunnel
``` 
**Step 10:** Verify  
Update your `/etc/hosts` file to add a DNS record to resolve `nginx.tmanet.com` to ingress gateway external IP
```bash
kubectl get services/istio-ingressgateway -n istio-system
```
Access https://nginx.tmanet.com
