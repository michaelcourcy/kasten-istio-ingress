# Goal 

Show how to expose kasten with an Istio ingress

# Install ISTIO

This guide was executed on EKS 1.32.9 and istio 1.28.2
```
Client Version: v1.34.2
Kustomize Version: v5.7.1
Server Version: v1.32.9-eks-3025e55
Warning: version difference between client (1.34) and server (1.32) exceeds the supported minor version skew of +/-1
client version: 1.28.2
control plane version: 1.28.2
data plane version: 1.28.2 (2 proxies)
```

1. Download and install istioctl if you haven't already
```
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
```

2. Make sure your GKE cluster is ready and your kubeconfig is configured

3. Install Istio with the demo profile (adjust as needed), we just want to work with the ingress so we don't install the CNI component
```
istioctl install --set profile=demo --set components.cni.enabled=false -y
```

4. Verify that Istio’s pods are running in the istio-system namespace
```
kubectl get pods -n istio-system
```

# Creating an ingress

Creating an ingress with Istio involves defining an Istio Gateway and a VirtualService. The Gateway resource maps external traffic to the cluster while the VirtualService directs the traffic to the appropriate service. For example, to expose a service called "kasten-service" on port 80, you could use the following YAML definitions:

```yaml
cat <<EOF | kubectl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kasten-gateway
  namespace: istio-system  # This ensures the resource is created in the kasten-io namespace
spec:
  selector:
    istio: ingressgateway  # Use Istio's default ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"  # Accept traffic for all hosts, or restrict as needed

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kasten-virtualservice
  namespace: kasten-io  # This ensures the resource is created in the kasten-io namespace
spec:
  hosts:
  - "*" # Must match the hosts in the Gateway
  gateways:
  - istio-system/kasten-gateway
  http:
  - match:
    - uri:
        prefix: /k10
    route:
    - destination:
        host: gateway  # Replace with your actual service name
        port:
          number: 80         # Replace if your service listens on a different port
EOF
```

Find out on which IP your ingressgateway is exposed
```
kubectl get svc -n istio-system
NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                                                      AGE
istio-egressgateway           ClusterIP      10.100.12.213    <none>                                                                    80/TCP,443/TCP                                                               30m
istio-ingressgateway          LoadBalancer   10.100.223.167   a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com   15021:30916/TCP,80:30418/TCP,443:32278/TCP,31400:31625/TCP,15443:31767/TCP   30m
istiod                        ClusterIP      10.100.54.95     <none>                                                                    15010/TCP,15012/TCP,443/TCP,15014/TCP                                        30m
istiod-revision-tag-default   ClusterIP      10.100.204.30    <none>                                                                    15010/TCP,15012/TCP,443/TCP,15014/TCP                                        30m
```

The ingressgateway is exposed on the public IP `a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com`

Use this IP in your browser 
```
http://a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com/k10/
```

# use a domain resolution instead 

Of course this solution is naive, all the request pointing to `a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com` are now redirected to the gateway kasten-io services.

You want your ingress gateway to handle multiple service. There is plenty way to do it but let's explore a simple solution : **domain resolution**.

I will create a A Record in my DNS provider (In my case it's AWS Route 53) to point to the the CNAME `a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com`. For instance for the wilcard domain `mcourcy-k10-on-istio.kastenevents.com`.

Let's check resolution work as expected 

```
dig mcourcy-k10-on-istio.kastenevents.com
...
;; ANSWER SECTION:
mcourcy-k10-on-istio.kastenevents.com. 60 IN CNAME a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com.
a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com. 60 IN A 3.210.76.197
a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com. 60 IN A 54.235.70.18
...
```

As you can read the resolution work as expected. 

Now try in your browser 
```
http://mcourcy-k10-on-istio.kastenevents.com/k10/ 
```

And you'll be redirected to Kasten. 

But that's not what we want. We don't want any domain that point to the public ip to be serving kasten.

Let' create another record that point to the samme CNAME 
```
dig mcourcy-otherapp.kastenevents.com
...
;; ANSWER SECTION:
mcourcy-otherapp.kastenevents.com. 60 IN CNAME  a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com.
a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com. 60 IN A 3.210.76.197
a2c5b47b085d44bc9a22504f0519e7a6-1692155039.us-east-1.elb.amazonaws.com. 60 IN A 54.235.70.18
```

And if I try in my browser 
```
http://mcourcy-otherapp.kastenevents.com/k10/ 
```

I'm also redirected to the Kasten UI. Which is not what I want, let's fix that by changing the gateway and virtual service 


```yaml
cat <<EOF | kubectl replace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kasten-gateway
  namespace: istio-system  # This ensures the resource is created in the kasten-io namespace
spec:
  selector:
    istio: ingressgateway  # Use Istio's default ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "mcourcy-k10-on-istio.kastenevents.com"  

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kasten-virtualservice
  namespace: kasten-io  # This ensures the resource is created in the kasten-io namespace
spec:
  hosts:
  - "mcourcy-k10-on-istio.kastenevents.com" # Must match the hosts in the Gateway
  gateways:
  - istio-system/kasten-gateway
  http:
  - match:
    - uri:
        prefix: /k10
    route:
    - destination:
        host: gateway  # Replace with your actual service name
        port:
          number: 80         # Replace if your service listens on a different port
EOF
```


Now it you try to access `http://mcourcy-k10-on-istio.kastenevents.com/k10/` it will direct you to the kasten UI but `http://mcourcy-otherapp.kastenevents.com/k10/` will be a 404.

# Use HTTPS instead of HTTP 

Of course we strongly advise that you use a TLS protocol for any services that need to traverse the internet. 

We're going to take 2 approaches https with a private CA (that you will generate in this example) and with a public CA using cert-manager with "Let's Encrypt".

## Using a private CA 

For the sake of this example we're going to generate our own CA.

```
# Step 1: Generate the CA private key and certificate
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1825 -out ca.crt \
  -subj "/CN=Kasten Demo CA/O=Kasten/C=US"

# Step 2: Generate the server private key
openssl genrsa -out tls.key 4096

# Step 3: Generate the server certificate signing request (CSR)
openssl req -new -key tls.key -out server.csr \
  -subj "/CN=mcourcy-k10-on-istio.kastenevents.com"

# Step 4: Create a config file for the certificate extensions
cat > server.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = mcourcy-k10-on-istio.kastenevents.com
EOF

# Step 5: Sign the server certificate with the CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out tls.crt -days 365 -sha256 -extfile server.ext

# Step 6: Create the Kubernetes secret in istio-system namespace
kubectl create secret tls kasten-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n istio-system

# Step 7: Save the CA certificate for distribution to users
echo "CA certificate saved in ca.crt - distribute this to users to install in their truststore"

# Step 8: Clean up temporary files (keep ca.crt for distribution)
rm server.csr server.ext ca.key ca.srl tls.key tls.crt
```

The ca.crt file is what users need to install in their browser/system truststore.

Then update your Gateway to use HTTPS:
```
cat <<EOF | kubectl replace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kasten-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: kasten-tls
    hosts:
    - "mcourcy-k10-on-istio.kastenevents.com"
EOF
```

Now you can try `https://mcourcy-k10-on-istio.kastenevents.com/k10/` and you'll have the usual warning message "This connection is not private ... " just click `visit website` and you get kasten.

## Using a public CA with certmanager and let's encrypt 

### Step 1: Install cert-manager

In order to generate proper certificates from Let's Encrypt, we need to install cert-manager.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Verify cert-manager is running:
```bash
kubectl get pods -n cert-manager
```

### Step 2: Create a Let's Encrypt ClusterIssuer 

Create a ClusterIssuer that will use Let's Encrypt to issue certificates. Replace the email with your own:

```bash
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: michael@kasten.io  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: istio
EOF
```

### Step 3: Create a Certificate resource

Create a Certificate resource that will automatically request and renew a certificate from Let's Encrypt:

```bash
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kasten-tls
  namespace: istio-system  # Must be in the same namespace as the Gateway
spec:
  secretName: kasten-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: mcourcy-k10-on-istio.kastenevents.com
  dnsNames:
  - mcourcy-k10-on-istio.kastenevents.com
EOF
```

Wait for the certificate to be issued (this may take a minute or two):
```bash
kubectl get certificate -n istio-system
kubectl describe certificate kasten-tls -n istio-system
```

### Step 4: Update the Gateway to use HTTPS

Once the certificate is ready, update your Gateway to listen on port 443 with TLS enabled:

```bash
cat <<EOF | kubectl replace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kasten-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: kasten-tls  # This matches the Certificate's secretName
    hosts:
    - "mcourcy-k10-on-istio.kastenevents.com"
EOF
```

Now you can access the Kasten UI with `https://mcourcy-k10-on-istio.kastenevents.com/k10/` with a valid Let's Encrypt certificate trusted by all browsers!

# Uninstall ISTIO

```
istioctl uninstall --purge -y
```

