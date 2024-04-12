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

### 3. Deploy VNFS and SFC-E2E-Collector

```bash
master@user:~/k8s$ kubectl create ns testbed

master@user:~/k8s$ helm repo add vnf-scc-sfc https://k8s-sfc-deployment.github.io/VNF-SCC-SFC
master@user:~/k8s$ helm install vnf-account-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/account.yaml
master@user:~/k8s$ helm install vnf-firewall-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/firewall.yaml
master@user:~/k8s$ helm install vnf-firewall-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/firewall.yaml
master@user:~/k8s$ helm install vnf-host-id-injection-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/host-id-injection.yaml
master@user:~/k8s$ helm install vnf-ids-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/ids.yaml
master@user:~/k8s$ helm install vnf-ids-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/ids.yaml
master@user:~/k8s$ helm install vnf-nat-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/nat.yaml
master@user:~/k8s$ helm install vnf-nat-1 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/nat.yaml
master@user:~/k8s$ helm install vnf-registry-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/registry.yaml
master@user:~/k8s$ helm install vnf-tcp-optimizer-0 vnf-scc-sfc/vnf-scc-sfc -n testbed -f vnfs/tcp-optimizer.yaml

master@user:~/k8s$ kubectl apply -f sfc-e2e-collector.yaml
```

### 4. Deploy Ingress-Nginx Baremetal

If you don't know about that, please read https://kubernetes.github.io/ingress-nginx/deploy/baremetal/.

```bash
master@user:~/k8s$ kubectl apply -f externals/ingress-nginx-baremetal.yaml
```

And, replace [`/k8s/ingress`](/k8s/ingress.yaml) `<Please Replace>` with domain.

```bash
master@user:~/k8s$ kubectl apply -f ingress.yaml
```

### 5. Check Result

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

Open [`/k8s/externals/prometheus/prometheus.yaml`](/k8s/externals/prometheus/prometheus.yaml) and fill below part to get node exporter information
```yaml
      # node-exporter
      - job_name: 'node-exporter'
        static_configs:
          - targets: # Please Fill this
            - <master-node-ip>:9100
            - <slave-node1-ip>:9100
            - <slave-node2-ip>:9100
```

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

### 6. [AutoScaler Metric Server] Metrics Server

- now I use metrics-server for horizontal auto scaling

```bash
master@user:~/k8s$ kubectl apply -f externals/metrics-server
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
