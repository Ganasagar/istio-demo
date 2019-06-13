# Setup Istio with MKE

## Installation

This installation guide was tested with the following components:

- Mesosphere DC/OS Enterprise 1.12.3
- Mesosphere Kubernetes Engine 2.2.0-1.13.3
- Edge-LB 1.3.0
- Istio 1.0.5

Download and unpack Istio to your workstation:

```bash
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.8 sh -
cd istio-1.1.8
```

Configure Istioctl to your workstation:

```bash
export PATH=$PWD/bin:$PATH
```

Add the helm repo for Istio to your workstation:

```bash
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.8/charts/

```

Deploy the custom resource and app definitions in Permissive Mode using`kubectl`:

```bash
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done

kubectl apply -f install/kubernetes/istio-demo.yaml
```

Deploy the custom resource and app definitions in Strict MTLS Mode using`kubectl`:

```bash
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done

kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```

Check that all pods and services in the newly created `istio-system` are in the state `running` or `completed`:

```bash
➜  istio-1.1.8 kubectl get po -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-67c69bb567-xfrxr                  1/1     Running     0          85s
istio-citadel-58fc8bfb55-6z7pk            1/1     Running     0          84s
istio-cleanup-secrets-1.1.8-27c88         0/1     Completed   0          91s
istio-egressgateway-6c4778d489-dmgzq      0/1     Running     0          85s
istio-galley-79595c6b67-lmjs7             1/1     Running     0          86s
istio-grafana-post-install-1.1.8-l9qw7    0/1     Completed   0          92s
istio-ingressgateway-5c85cfc4c6-fkgl5     0/1     Running     0          85s
istio-pilot-7cc5b9d8b-r6mtq               1/2     Running     0          85s
istio-policy-66cccbdd69-2c28d             2/2     Running     2          85s
istio-policy-66cccbdd69-tsrwm             2/2     Running     0          22s
istio-security-post-install-1.1.8-gqlc4   0/1     Completed   0          90s
istio-sidecar-injector-6fd47fb7c8-n6zdz   1/1     Running     0          84s
istio-telemetry-74595b8b7-8jfpx           2/2     Running     2          85s
istio-tracing-5d8f57c8ff-zqh92            1/1     Running     0          84s
kiali-d4d886dd7-jtvc2                     1/1     Running     0          85s
prometheus-d8d46c5b5-l5srp                1/1     Running     0          84s
```

The Istio deployment also created a couple of services you can list them via:

```bash
kubectl get svc -n istio-system
```

## Visualizing your Mesh with Kiali

Optionally you can also install the Kiali add-on for Istio and use the web-based graphical user interface to view service graphs of the mesh and your Istio configuration objects. Additionally, you use the Kiali Public API to generate graph data in the form of consumable JSON.

Documentation: [Istio - Kilali][istio-kiali]

```bash
KIALI_USERNAME=`echo -n "admin" | base64`
KIALI_PASSPHRASE=`echo -n "admin" | base64`
NAMESPACE=istio-system
```

Now, create a secret for these credentials:

```yaml
cat <<EOF | tee /tmp/kiali-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: $NAMESPACE
  labels:
    app: kiali
type: Opaque
data:
  username: $KIALI_USERNAME
  passphrase: $KIALI_PASSPHRASE
EOF

kubectl apply -f /tmp/kiali-secrets.yaml
```

Edit the Istio configuration via helm template and apply the new Istio configuration:

```bash
helm template \
    --set kiali.enabled=true \
    --set "kiali.dashboard.jaegerURL=http://$(kubectl get svc tracing -n istio-system -o jsonpath='{.spec.clusterIP}'):80" \
    --set "kiali.dashboard.grafanaURL=http://$(kubectl get svc grafana -n istio-system -o jsonpath='{.spec.clusterIP}'):3000" \
    install/kubernetes/helm/istio \
    --name istio --namespace istio-system > /tmp/istio.yaml

kubectl apply -f /tmp/istio.yaml
```

## Port-Forwarding of Services

Without the need to expose each service, you can just do a port-forward for a quick test. In the following example, you access the port of Kiali directly. So you are able to view the web interface in the browser at [http://localhost:20001](http://localhost:20001).

```bash
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') \
  20001:20001
```

You can do the amse for other services like Grafana, Prometheus or Jager:
To port-forward Grafana:

```bash
# Grafana
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana \
  -o jsonpath='{.items[0].metadata.name}') 3000:3000

# Prometheus
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') \
  9090:9090

# Service Graph
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') \
  8088:8088

# Tracing/Jaeger
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{.items[0].metadata.name}') \
  16686:16686
```

## Expose Istio Gateway via Edge-LB

Istio will also deploy an ingress gateway, you can list it via the following command:

```bash
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                   AGE
istio-ingressgateway   LoadBalancer   10.100.176.216   <pending>     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32543/TCP,8060:30311/TCP,853:32330/TCP,15030:31962/TCP,15031:31933/TCP   4h10m
```

The Istio gateway for HTTP port `80` listens on NodePort `31380`, so we will setup an Edge-LB frontend that binds to port `80` and points to a backend defined for Mesos tasks with the name pattern `kube-node-.*`, the framework name `kubernetes-cluster` (default name), and destination port `31380`. The full Edge-LB pool configuration looks like this:

```json
cat <<EOF | tee /tmp/pool-istio-ingressgateway.json
{
  "apiVersion": "V2",
  "name": "istio-ingressgateway",
  "cpus": 0.7,
  "count": 1,
  "haproxy": {
    "frontends": [
      {
        "bindPort": 80,
        "protocol": "TCP",
        "linkBackend": {
          "defaultBackend": "kube-node-http"
        }
      },
      {
        "bindPort": 443,
        "protocol": "TCP",
        "linkBackend": {
          "defaultBackend": "kube-node-https"
        }
      }
    ],
    "backends": [{
      "name": "kube-node-http",
      "protocol": "TCP",
      "services": [{
        "mesos": {
          "frameworkName": "kubernetes-cluster1",
          "taskNamePattern": "^kube-node-.*$"
        },
        "endpoint": {
          "port": 31380,
          "type": "AUTO_IP"
        }
      }]
    },{
      "name": "kube-node-https",
      "protocol": "TCP",
      "services": [{
        "mesos": {
          "frameworkName": "kubernetes-cluster1",
          "taskNamePattern": "^kube-node-.*$"
        },
        "endpoint": {
          "port": 31390,
          "type": "AUTO_IP"
        }
      }]
    }],
    "stats": {
      "bindPort": 9090
    }
  }
}
EOF
```

In the next step, we will give the service principal for Edge-LB the permissions to control the pool istio-ingressgateway and apply the pool configuration.

```bash
dcos security org users grant edge-lb-principal dcos:adminrouter:service:dcos-edgelb/pools/istio-ingressgateway full
dcos edgelb create /tmp/pool-istio-ingressgateway.json
```

**Note:** The Edge-LB pool will not serve any response on port `80` until a Istio gateway was deployed. This will be done as part of the demo in the following section:

- [Demo: BookInfo](../bookinfo/)

[istio-kiali]: https://istio.io/docs/tasks/telemetry/kiali/
