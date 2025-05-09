# istio-ambient-gke-vcluster-demo
This repo illustrates how to install Istio in **ambient** mode on GKE cluster and test traffic splitting.

### 1. Install istio 
```
kubectl create ns istio-system
kubectl apply -f gcp-critical-pods.yaml
istioctl install --set profile=ambient --set values.global.platform=gke -f istio-operator.yaml
```
### 2. Install Kubernetes Gateway API CRDs
Note that the Kubernetes Gateway API CRDs do not come installed by default on most Kubernetes clusters, so make sure they are installed before using the Gateway API:
```
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0-rc.1/standard-install.yaml
```
### 3. Label namespace and create Waypoint Proxy Gateway
```
kubectl label ns default istio.io/dataplane-mode=ambient
kubectl apply -f waypoint-gateway.yaml
```
### 4. Deploy application, service, configmaps and istio resources
```
kubectl apply -f  nginx-deployments.yaml
kubectl apply -f nginx-index-configmaps.yaml
kubectl apply -f nginx-service.yaml
istio-resources.yaml
```
### 5. Get external ip of istio-ingressgateway
```
k get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   <ip>             <external-ip>    <>                                           3h41m

```
### 6. Access Application
```
for i in {1..10}; do curl http://<external-ip>/; done
```
### 7. Traffic shifting
To route traffic to only v1:
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: nginx-virtualservice
spec:
  hosts:
    - "*"
  gateways:
    - my-ingress-gateway
  http:
    - route:
        - destination:
            host: nginx-service.default.svc.cluster.local
            subset: v1
          weight: 100
        - destination:
            host: nginx-service.default.svc.cluster.local
            subset: v2
          weight: 0
EOF

```
