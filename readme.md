# Goal 

Show how to expose kasten with an Istio ingress

# Install ISTIO

This guide was executed on GKE 
```
kubectl version  
Client Version: v1.30.3
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.31.6-gke.1064001
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
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kasten-gateway
  namespace: kasten-io  # This ensures the resource is created in the kasten-io namespace
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
  - kasten-gateway
  http:
  - route:
    - destination:
        host: kasten-service  # Replace with your actual service name
        port:
          number: 80         # Replace if your service listens on a different port
```

Find out on which IP your ingressgateway is exposed
```
kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.2.14.71   <none>         80/TCP,443/TCP                                                               7m18s
istio-ingressgateway   LoadBalancer   10.2.10.68   34.121.37.12   15021:30850/TCP,80:32692/TCP,443:30245/TCP,31400:30316/TCP,15443:32099/TCP   7m18s
istiod                 ClusterIP      10.2.5.117   <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP                                        7m28s
```

The ingressgateway is exposed on the public IP `34.121.37.12`

Use this IP in your browser 
```
http://34.121.37.12/k10/
```

Happy backup !

# Next TODO  

- use https
- use a specific domain 


# Uninstall ISTIO

```
istioctl uninstall --purge -y
```