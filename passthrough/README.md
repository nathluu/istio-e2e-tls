## Procedure
**Step 1:** Generate certificates and keys  
```bash
#nginx red server
openssl req -out red.tmanet.com.csr -newkey rsa:2048 -nodes -keyout red.tmanet.com.key -subj "/CN=red.tmanet.com/O=DC"
openssl x509 -req -days 365 -CA tmanet.com.crt -CAkey tmanet.com.key -set_serial 1 -in red.tmanet.com.csr -out red.tmanet.com.crt
```
**Step 2:** Create configmap and secret for nginx red server  
```bash
kubectl create -n apps configmap nginx-configmap-red --from-file=nginx.conf=./nginx.conf --from-file=index.html=./index.html
# kubectl create -n apps configmap nginx-configmap-red --from-file=nginx.conf=./nginx-mutual-tls.conf # mTLS enabled
kubectl create -n apps secret generic nginx-server-certs-red --from-file=tls.key=red.tmanet.com.key \
--from-file=tls.crt=red.tmanet.com.crt --from-file=ca.crt=tmanet.com.crt
```
**Step 4:** Create nginx deployment
```bash
kubectl apply -n apps -f apps.yaml
```
**Step 5:** Verify  
Update your `/etc/hosts` file to add DNS record to resolve `red.tmanet.com` to ingress gateway external IP
```bash
kubectl get services/istio-ingressgateway -n istio-system
```
Access https://red.tmanet.com
curl https://red.tmanet.com/ -k --cert client.gateway.crt --key client.gateway.key
