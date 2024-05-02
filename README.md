# K8S Deploy Example

- [x] test k8s deployment
- [x] apply monitoring
  - [x] grafana
  - [x] prometheus
  - [x] prometheus exporter
    - [x] cilium / hubble
    - [x] node-exporter
    - [x] kepler

## Run

- kubernetes version : v1.29.3
- helm version : v3.14.3
- cilium cli : v0.16.3

### 1. Build Cluster

below is baremetal environment.

```bash
master@user:~$ kubeadm init
```


```bash
slave@user:~$ kubeadm join
```

### 2 CNI setup (Cilium)

```bash
master@user:~$ helm repo add cilium https://helm.cilium.io/
master@user:~/k8s$ helm install cilium cilium/cilium --version 1.15.2 -n kube-system -f externals/cilium/values.yaml
```

## 3. Service Mesh (Istio)

```bash
master@user:~/k8s$ helm install istio-base istio/base -n istio-system -f externals/istio/base-values.yaml --create-namespace --version 1.21.1
master@user:~/k8s$ helm install istiod istio/istiod -n istio-system -f externals/istio/istiod-values.yaml --version 1.21.1
master@user:~/k8s$ helm install istio-ingressgateway istio/gateway  -n istio-system -f externals/istio/ingressgateway-value.yaml --version 1.21.1

master@user:~/k8s$ kubectl create ns testbed
master@user:~/k8s$ kubectl label ns testbed istio-injection=enabled
```

### 4. Deploy VNFS and SFC-E2E-Collector

```bash
master@user:~/k8s$ helm repo add vnf-scc-sfc https://k8s-sfc-deployment.github.io/VNF-SCC-SFC

master@user:~/k8s$ helm install vnf-account-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/account.yaml --set nameOverride=vnf-account-0
master@user:~/k8s$ helm install vnf-firewall-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/firewall.yaml --set nameOverride=vnf-firewall-0
master@user:~/k8s$ helm install vnf-firewall-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/firewall.yaml --set nameOverride=vnf-firewall-1
master@user:~/k8s$ helm install vnf-host-id-injection-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/host-id-injection.yaml --set nameOverride=vnf-host-id-injection-0
master@user:~/k8s$ helm install vnf-ids-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/ids.yaml --set nameOverride=vnf-ids-0
master@user:~/k8s$ helm install vnf-ids-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/ids.yaml --set nameOverride=vnf-ids-1
master@user:~/k8s$ helm install vnf-nat-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/nat.yaml --set nameOverride=vnf-nat-0
master@user:~/k8s$ helm install vnf-nat-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/nat.yaml --set nameOverride=vnf-nat-1
master@user:~/k8s$ helm install vnf-registry-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/registry.yaml --set nameOverride=vnf-registry-0
master@user:~/k8s$ helm install vnf-tcp-optimizer-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/tcp-optimizer.yaml --set nameOverride=vnf-tcp-optimizer-0

master@user:~/k8s$ helm repo add sfc-e2e-collector https://k8s-sfc-deployment.github.io/SFC-E2E-collector
master@user:~/k8s$ helm install sfc-e2e-collector sfc-e2e-collector/sfc-e2e-collector -n testbed -f sfc-e2e-collector/value.yaml
```

### 5. Deploy Ingress-Nginx Baremetal

If you don't know about that, please read https://kubernetes.github.io/ingress-nginx/deploy/baremetal/.

```bash
master@user:~/k8s$ kubectl apply -f externals/ingress-nginx-baremetal.yaml
```

And, replace [`/k8s/ingress`](/k8s/ingress.yaml) `<Please Replace>` with domain.

```bash
master@user:~/k8s$ kubectl apply -f ingress.yaml
```

### 6. Check Result

```bash
# check ingress-nginx-controller's ports(<http-port>, and <https-port>)
master@user:~/k8s$ kubectl get svc -n ingress-nginx 
# NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                   AGE
# ingress-nginx-controller             NodePort    10.109.147.179   <none>        80:<http-port>/TCP,443:<https-port>/TCP   123m
# ingress-nginx-controller-admission   ClusterIP   10.103.10.235    <none>        443/TCP                                   123m

# check <host> and <your-ip>
master@user:~/k8s$ kubectl get ing -n testbed
# NAME      CLASS   HOSTS    ADDRESS     PORTS   AGE
# ingress   nginx   <host>   <your-ip>   80      121m

# call
master@user:~/k8s$ curl http://<host>:<http-port>/ids/openapi.json
master@user:~/k8s$ curl http://<host>:<http-port>/firewall/openapi.json
```

## Monitoring

### 1. Grafana

```bash
master@user:~/k8s$ kubectl create ns monitoring
master@user:~/k8s$ kubectl apply -f externals/grafana
```

### 2. Prometheus

```bash
master@user:~/k8s$ kubectl apply -f externals/prometheus
```

### 3. [Prometheus Exporter 1] Cilium / Hubble

```bash
master@user:~/k8s$ cilium hubble enable --ui
master@user:~/k8s$ cilium status # wait until hubble-ui wake up
```

### 4. [Prometheus Exporter 2] Node Exporter

```bash
master@user:~/k8s$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
master@user:~/k8s$ helm install -n node-exporter --create-namespace --version 1.7.0 -f externals/node-exporter/value.yaml
```

### 5. [Prometheus Exporter 3] Kepler

```bash
master@user:~/k8s$ helm repo add kepler https://sustainable-computing-io.github.io/kepler-helm-chart
master@user:~/k8s$ helm install kepler kepler/kepler -n kepler --create-namespace --version release-0.7.8 -f externals/kepler/value.yaml
```

### 6. [Prometheus Exporter 4] Custom Exporter - NMBN Exporter

Open [`/k8s/nmbn-exporter/value.yaml`](/k8s/nmbn-exporter/value.yaml) and fill below part to get node exporter information

```yaml
targets:
- ip: <target ip1>
- io: <target ip2>
```

```bash
master@user:~/k8s$ helm repo add nmbn-exporter https://k8s-sfc-deployment.github.io/nmbn-exporter
master@user:~/k8s$ helm install nmbn-exporter nmbn-exporter/nmbn-exporter -n nmbn-exporter -f nmbn-exporter/value.yaml
```

### 7. [AutoScaler Metric Server] Metrics Server

- now I use metrics-server for horizontal auto scaling

```bash
master@user:~/k8s$ kubectl apply -f externals/metrics-server
```

### 8. [Optional] ISTIO Addon

you also can use istio addon, this repo have [`/k8s/externals/istio/addons`](/k8s/externals/istio/addons).

```bash
master@user:~/k8s$ kubectl apply -f externals/externals/istio/addons

master@user:~/k8s$ istioctl dashboard jaeger
master@user:~/k8s$ istioctl dashboard grafana
master@user:~/k8s$ istioctl dashboard kiali
master@user:~/k8s$ istioctl dashboard loki
master@user:~/k8s$ istioctl dashboard prometheus
```

### Monitoring Service url

- Hubble UI  
  http://\<ingress-host\>:\<nodePort-http-port\>/
- Grafana  
  http://\<ingress-host\>:\<nodePort-http-port\>/grafana
- Prometheus  
  http://\<ingress-host\>:\<nodePort-http-port\>/prometheus/graph
- Firewall API  
  http://\<ingress-host\>:\<nodePort-http-port\>/firewall/docs
- IDS API  
  http://\<ingress-host\>:\<nodePort-http-port\>/ids/docs


## Externals

- `ingress-nginx`: v1.10.0
- `kubernetes-dashboard`: v2.7.0
- `cilium`: v1.15.2
- `node-exporter`: 1.7.0
- `kepler`: release-0.7.8
