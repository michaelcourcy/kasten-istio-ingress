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


# Managing authorization policy 

Istio AuthorizationPolicy is a security resource that enables access control for workloads in the service mesh. It allows you to define who can access what services and under what conditions.

Often zero trust architecture define a deny all AuthorizationPolicy, which make security audit much easier. 

Let's create one 
```
cat <<EOF | kubectl create -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all-by-default
  namespace: istio-system
spec:
  action: ALLOW
  rules: []  # Empty = deny all traffic mesh-wide
EOF
```

If you reload the page you'll get 
```
RBAC: access denied
```

This is a 403 error meaning "I know who you are (or don't care), but you don't have permission to access this resource", The request reaches Envoy proxy, policy is evaluated, and access is denied.

This rule is mesh wide because defined in the istio-system. It will block any communication for pods belonging to the service mesh. 

Since the kasten pods are not in the service mesh (no sidecar injection) inter-namespace communication in kasten-io is ok. 

But since the ingress gateway is using the envoy proxy it will block the access to the kasten UI. In order to solve that you need to create an extra Authorization policy for kasten.

```
cat <<EOF | kubectl create -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-k10-ingress-gateway
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway  # Only applies to the ingress gateway pods
  action: ALLOW
  rules:
  - to:
    - operation:
        paths: ["/k10/*"]
        hosts: ["mcourcy-k10-on-istio.kastenevents.com"]
EOF
```

Now you can laod the UI again. 

# Uninstall ISTIO

```
istioctl uninstall --purge -y
```

# Access through tunneling and port-forward in a jump host

When you don't have any ingress solution (Load balancer, istio ingress, nginx ingress, traefik ...) you can use a port forward command.

## Direct port-forward (when you have direct cluster access)

If you have direct access to the cluster from your local machine:

```bash
kubectl --namespace kasten-io port-forward service/gateway 8080:80
```

It opens locally the port 8080 and the Kasten dashboard will be available at http://127.0.0.1:8080/k10/#/.

## SSH Tunneling (when accessing through a jump host)

But if you access the cluster through a jumphost/bastion via SSH without any graphical interface, you won't be able to open a browser on the jumphost.

In order to access the dashboard from your laptop (the one from which you initiate the SSH session), you can use SSH tunneling.

### Step 1: Start port-forward on the jump host

First, SSH into your jump host and start the port-forward:

```bash
ssh user@jumphost
kubectl --namespace kasten-io port-forward service/gateway 8080:80
```

Keep this terminal session open.

### Step 2: Create an SSH tunnel from your laptop

Open a **new terminal on your laptop** and create an SSH tunnel that forwards a local port to the jump host's port 8080:

```bash
ssh -L 8080:localhost:8080 user@jumphost
```

This command:
- `-L 8080:localhost:8080` - Creates a local port forward
- First `8080` - Port on your laptop
- `localhost:8080` - Port on the jumphost (where kubectl port-forward is listening)

Keep this SSH session open as well.

### Step 3: Access Kasten from your laptop browser

Now open your browser on your laptop and navigate to:

```
http://localhost:8080/k10/#/
```

The traffic flow is:
```
Your Laptop (port 8080) → SSH Tunnel → Jump Host (port 8080) → kubectl port-forward → Kubernetes Service (gateway)
```

## Alternative: Single command with SSH tunnel

You can also combine both commands in a single SSH session using background processes:

```bash
ssh -L 8080:localhost:8080 user@jumphost "kubectl --namespace kasten-io port-forward service/gateway 8080:80"
```

This will:
1. Create the SSH tunnel
2. Execute the kubectl port-forward command on the jump host
3. Keep both running until you press Ctrl+C

Then access http://localhost:8080/k10/#/ in your browser.

## If you are using PuTTY instead of plain SSH

Windows users often use PuTTY as their SSH client. Here's how to set up the SSH tunnel using PuTTY:

### Step 1: Configure the SSH connection

1. Open PuTTY
2. In the **Session** category:
   - Enter your jump host address in **Host Name (or IP address)**: `jumphost` or the IP address
   - Port: `22`
   - Connection type: `SSH`

### Step 2: Configure the tunnel

1. In the left panel, navigate to: **Connection** → **SSH** → **Tunnels**
2. Configure the port forwarding:
   - **Source port**: `8080` (port on your local Windows machine)
   - **Destination**: `localhost:8080` (port on the jump host)
   - Select **Local**
   - Click **Add**
3. You should see `L8080  localhost:8080` appear in the **Forwarded ports** list

### Step 3: Save the session (optional)

1. Go back to **Session** category
2. Enter a name in **Saved Sessions** (e.g., "Jumphost Tunnel")
3. Click **Save**

### Step 4: Connect

1. Click **Open**
2. Log in to the jump host with your credentials
3. Once connected, run the port-forward command in the PuTTY terminal:
   ```bash
   kubectl --namespace kasten-io port-forward service/gateway 8080:80
   ```
4. Keep the PuTTY window open

### Step 5: Access Kasten

Open your browser on Windows and navigate to:
```
http://localhost:8080/k10/#/
```

**Note**: If you need to forward multiple ports, repeat Step 2 for each port with different source and destination port numbers.

## Troubleshooting

**Port already in use:**
If port 8080 is already in use on your laptop, change it:
```bash
ssh -L 9090:localhost:8080 user@jumphost
# Then access http://localhost:9090/k10/#/
```

**Connection refused:**
- Make sure the kubectl port-forward is still running on the jump host
- Verify you can reach the jump host: `ping jumphost` or `ssh user@jumphost`
- Check if the gateway service exists: `kubectl get svc -n kasten-io`

**Cannot connect to Kubernetes from jump host:**
- Verify kubeconfig is properly set on the jump host: `kubectl cluster-info`
- Check if you have proper permissions: `kubectl auth can-i get pods -n kasten-io`


