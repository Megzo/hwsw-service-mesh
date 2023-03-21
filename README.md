# HWSW Service Mesh Meetup Demo

This is a short demo of service mesh that was presented during my HWSW meetup.
YouTube link: to be added.
    
## Base Kubernetes setup

All tests were run on a hand-built 3 node Kubernetes cluster:
 - K8s version: 1.26.1
 - 3 VMs in GKE, all are masters and workers at the same time
 - no `LoadBalancer` provider, `ExternalIPs` are added manually
 - Nginx Ingress and Cert Manager are already installed

## Install the sample appliaction without Istio

Install the [Bookinfo](https://istio.io/latest/docs/examples/bookinfo/) application:
```
kubectl apply -f bookinfo.yaml
kubectl get pod,svc
```

Add an Ingress via Nginx:
```
kubectl apply -f ingress.yaml
```


## Installing Istio on Kubernetes with `istioctl`

(Other option would be helm since the operator install is deprecated):

- helm install: https://istio.io/latest/docs/setup/install/helm/

1.  Install Istio with demo profile

Based on https://istio.io/latest/docs/setup/getting-started/

Download Istio 1.16.1 for the x86_64 architecture:
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.16.1 TARGET_ARCH=x86_64 sh -
sudo install ./istio-1.16.1/bin/istioctl /usr/bin/
```

Trying it out:
```
istioctl -h
source <(istioctl completion bash)
echo 'source <(istioctl completion bash)' >>~/.bashrc
```

Install Istio (demo profile)
```
istioctl x precheck
istioctl install --set profile=demo -y
kubectl get pods -n istio-system
kubectl get crds | grep istio
```

Expose the Istio Ingress Gateway (in normal cluster you should get an external IP via the LoadBalancer provider):
```
# set your ExternalIP
export MYEXTERNALIP=10.166.0.48
kubectl -n istio-system patch svc/istio-ingressgateway -p "{\"spec\":{\"externalIPs\":[\"$MYEXTERNALIP\"]}}" 
```

## Put the bookinfo app into the mesh

Option 1: use `istioctl kube-inject`:
```
istioctl kube-inject -f bookinfo.yaml
istioctl kube-inject -f bookinfo.yaml | kubectl apply -f -
```

Option 2: inject the namespace and restart the pods:
```
kubectl label namespace default istio-injection=enabled
kubectl delete pods --all
watch kubectl get pods
```

## Installing istio addons (prometheus, grafana, kiali and others)
```
git clone https://github.com/istio/istio/
kubectl apply -f istio/samples/addons/
kubectl get pods,svc -n istio-system 
```

You can access the services via `kubectl port-forward`, e.g.
```
kubectl port-forward -n istio-system service/grafana 8080:3000 --address=0.0.0.0
```

Or use `NodePort`:
```
kubectl patch service -n istio-system grafana -p '{"spec":{"type": "NodePort"}}'
kubectl patch service -n istio-system kiali -p '{"spec":{"type": "NodePort"}}'
kubectl patch service -n istio-system prometheus -p '{"spec":{"type": "NodePort"}}'
kubectl patch service -n istio-system tracing -p '{"spec":{"type": "NodePort"}}'
kubectl get service -n istio-system
```

## Gateway, VirtualService and DestinationRule

Now let's publish the Bookinfo app via the Istion Ingress Gateway.
For that, you'll need to define a `Gateway` and a `VirtualService` for the `productpage` service:
```
kubectl apply -f gateway.yaml
```

Try to access the service via the Public IP of the Istio Ingress Gatewy.

Now, we can define the internal routing via the Istio features as well.
Well need `DestinationRules` and `VirtualServices` for that.
```
kubectl apply -f destenation-rule.yaml
kubectl apply -f virtual-service.yaml
```

Notice that the `reviews` service only uses the `v1` subset, so we'll only see that.

If we want to change for multiple subsets, we can use the following config:
```
kubectl apply -f reviews-50-50.yaml
```

A canary deployment would use someting similar, but with a small weight on the new version of the service:
```
kubectl apply -f reviews-95-5.yaml
```

The other interesting feature is that we can route based on HTTP header information.
So if we want to have a `test` user that will use the new version, try the following:
```
kubectl apply -f reviews-dark.yaml
```

Log in on the UI and see what happens.

## Clean-up

```
kubectl delete -f .
istioctl uninstall --purge
kubectl delete namespace istio-system
kubectl label namespaces default istio-injection-
```
