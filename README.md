# OpenTelemetry
OpenTelemetry-without-OneAgent

1. **Application code (instrumented with OpenTelemetry SDK)**:
Each microservice (frontend, cartservice, adservice, etc.) has OpenTelemetry instrumentation baked into its code/image. As requests happen, this SDK automatically generates spans — each span represents one unit of work (an HTTP call, a gRPC call, a DB query, etc.) with a start time, duration, and metadata (status code, endpoint, etc.).

2. **Spans** → **OpenTelemetry Collector**
Each service sends its spans over the network (via OTLP protocol, gRPC or HTTP) to the OpenTelemetry Collector you deployed as a daemonset. That's why you saw the collector receiving on ports 4317 (gRPC) and 4318 (HTTP) in its startup logs earlier.

3. **Collector** → **Dynatrace**
The Collector batches, processes, and forwards those spans onward via the otlphttp exporter — this is the piece you configured with your Dynatrace tenant URL and API token. The collector pushes data to Dynatrace's OTLP ingest endpoint.

4. **Dynatrace ingests and correlates**
Dynatrace receives the raw spans, stitches related spans together (using the traceparent header/trace ID that ties spans across services into one request), and builds the full distributed trace — which is why you can see a request flow from frontend → cartservice → checkoutservice, etc.

5. **Distributed Traces UI**:
That's the Explorer view you screenshotted — Dynatrace visualizes:


**Flow from Code:**

App code (OTel SDK) → generates spans → OTel Collector (receives, batches) → exports via OTLP → Dynatrace ingest → Distributed Traces UI

<img width="1703" height="927" alt="image" src="https://github.com/user-attachments/assets/e593d87a-bfb8-429c-b841-e84687b99915" />

# Observability Clinic — OpenTelemetry Operator (Minikube Edition)

This is an adapted version of the [Observability Clinic: OpenTelemetry without OneAgent](https://github.com/dynatrace-perfclinics/Observability-clinic---openTelemetry-without-OneAgent) tutorial, updated to run on **Minikube** (Docker driver, macOS Apple Silicon) instead of GKE. It deploys Google's **Online Boutique** demo app, instrumented with OpenTelemetry, and ships traces to **Dynatrace** without using OneAgent.

<p align="center"><img src="/image/opentelemetry.png" width="20%" alt="OpenTelemetry Logo" /></p>

## Prerequisites

- macOS (Apple Silicon or Intel)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — with Rosetta installed if prompted
- `jq`, `kubectl`, `git`, `helm`
- `minikube` (darwin-arm64 build for Apple Silicon)
- A Dynatrace tenant — [start a free trial](https://www.dynatrace.com/trial/) if you don't have one
- A Dynatrace API token with scope **Ingest OpenTelemetry traces** (generate under **Access Tokens** in the left menu)

### Recommended Docker Desktop resources
Allocate at least **10 GB memory** and **6 CPUs** to Docker Desktop (Settings → Resources) before starting minikube — the demo app, OTel Collector, and Prometheus stack together need real headroom. On a 16GB Mac this leaves enough for macOS itself.

---

## 1. Install and configure tools

```bash
brew install jq kubectl git helm
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

If Docker isn't installed yet:
```bash
brew install --cask docker
open -a Docker
```
Open Docker Desktop once, allow the Rosetta install if prompted, and wait for the whale icon in the menu bar to go steady. Then set the driver and confirm Docker is healthy:
```bash
minikube config set driver docker
docker version
```

## 2. Start the Minikube cluster

```bash
minikube start --nodes=2 --cpus=6 --memory=9000
kubectl get nodes
```
You should see 2 nodes in `Ready` status.

> If `minikube start` fails with a driver error, install Docker Desktop first (see above) — minikube needs a working driver (Docker, vfkit, etc.) to run.
>
> If it fails complaining Docker doesn't have enough memory, raise Docker Desktop's allocation under **Settings → Resources** and retry.

## 3. Clone the repository

```bash
git clone https://github.com/isItObservable/OpenTelemetryOperator
cd OpenTelemetryOperator
```

## 4. Enable Ingress

Unlike GKE, Minikube ships ingress-nginx as a built-in addon — no manifest needed.

```bash
minikube addons enable ingress
```

In a **separate terminal window** (leave it running for the entire session):
```bash
minikube tunnel
```
It will prompt for your **sudo password** to bind privileged ports 80/443 — enter it and leave this terminal open.

Minikube's ingress-nginx service defaults to `NodePort`, but the tutorial's manifests assume a real IP on port 80/443. Patch it to `LoadBalancer` so the tunnel can bind it:
```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n ingress-nginx
```
Wait until `EXTERNAL-IP` shows `127.0.0.1`.

## 5. Get the ingress IP and patch the app manifest

```bash
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
echo $IP   # should print 127.0.0.1
sed -i '' "s,IP_TO_REPLACE,$IP," hipster-shop-otel/k8s-manifest.yaml
```
> Note the `sed -i ''` syntax (with an empty string argument) — macOS `sed` requires this; Linux does not.

## 6. Deploy the Prometheus Operator

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```
(Skip the `nodeSelector` flags from the original GKE tutorial — they target dedicated observability nodes not present on a local minikube cluster.)

Enable the `remote-write-receiver` feature (needed for K6 metrics):
```bash
kubectl get prometheus
kubectl patch prometheus prometheus-kube-prometheus-prometheus \
  --type=merge -p '{"spec":{"enableFeatures":["remote-write-receiver"]}}'
kubectl get prometheus prometheus-kube-prometheus-prometheus -o yaml | grep -A2 enableFeatures
```

> On a resource-constrained cluster, Grafana and Alertmanager can be scaled down since they aren't required to prove tracing works:
> ```bash
> kubectl scale deployment prometheus-grafana --replicas=0
> kubectl patch alertmanager prometheus-kube-prometheus-alertmanager --type=merge -p '{"spec":{"replicas":0}}'
> ```
> (Alertmanager is managed by a CRD via the Prometheus Operator — scaling the StatefulSet directly gets reverted; patch the `Alertmanager` object instead.)

## 7. Deploy cert-manager and the OpenTelemetry Operator

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
kubectl get pods -n cert-manager
```
Wait until **all three** cert-manager pods show `1/1 Running` — this can take 30-60 seconds. Applying the operator manifest before cert-manager is ready causes a webhook connection error.

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
kubectl get pods -n opentelemetry-operator-system
```

## 8. Configure the OpenTelemetry Collector for Dynatrace

Get your tenant URL and API token:
- Tenant URL format: `https://<TENANTID>.live.dynatrace.com` (the classic ingest domain — note this differs from the newer `apps.dynatrace.com` Platform UI URL, though the tenant ID is the same)
- API token: Dynatrace → **Access Tokens** → generate new → scope **Ingest OpenTelemetry traces**

```bash
CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
export DT_TENANT_URL=https://<TENANTID>.live.dynatrace.com
export DT_API_TOKEN=<your-api-token>

sed -i '' "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," openTelemetry/openTelemetry-manifest_daemonset.yaml
sed -i '' "s,TENANTURL_TOREPLACE,$DT_TENANT_URL," openTelemetry/openTelemetry-manifest_daemonset.yaml
sed -i '' "s,DT_API_TOKEN_TO_REPLACE,$DT_API_TOKEN," openTelemetry/openTelemetry-manifest_daemonset.yaml
```

> **Known fix required:** newer OpenTelemetry Collector images (v0.15x+) removed the deprecated `logging` exporter in favor of `debug`. If the collector manifest still references `logging`, the pod will crash-loop with:
> `'exporters' the logging exporter has been deprecated, use the debug exporter instead`
>
> Fix it before applying:
> ```bash
> sed -i '' 's/logging:/debug:/' openTelemetry/openTelemetry-manifest_daemonset.yaml
> sed -i '' 's/\[logging,otlphttp\]/[debug,otlphttp]/' openTelemetry/openTelemetry-manifest_daemonset.yaml
> ```

Deploy the collector:
```bash
kubectl apply -f openTelemetry/openTelemetry-manifest_daemonset.yaml
kubectl get pods | grep otel
kubectl logs <collector-pod-name> --tail=50
```
Confirm the log shows `Everything is ready. Begin running and processing data.` with no export/auth errors.

## 9. Deploy the demo app

```bash
kubectl apply -f hipster-shop-otel/k8s-manifest.yaml
kubectl get pods
```
Wait for all ~11 microservices to reach `1/1 Running`. This app has 11 separate deployments so it can take 1-2 minutes.

> **Known fix — `adservice` (Java) may crash-loop:**
> - **OOMKilled**: the default 300Mi memory limit is too low for a JVM workload. Raise it:
>   ```bash
>   kubectl set resources deployment adservice --limits=memory=500Mi,cpu=300m --requests=memory=250Mi,cpu=200m
>   ```
> - **Liveness/readiness probe timeouts** (`grpc_health_probe timed out after 1s`) under CPU pressure: increase the probe timeout:
>   ```bash
>   kubectl patch deployment adservice --type=json -p='[
>     {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/timeoutSeconds","value":5},
>     {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/timeoutSeconds","value":5}
>   ]'
>   ```

## 10. Generate traffic

`k6loadgenerator` runs automatically as part of the app and generates continuous background traffic. You can also browse manually:
```bash
open http://onlineboutique.127.0.0.1.nip.io/
```
Click around, add items to the cart, and complete checkout to generate a rich multi-service trace.

Sanity-check the endpoint responds and returns a `traceparent` header:
```bash
curl -v http://onlineboutique.127.0.0.1.nip.io/
```

## 11. View traces in Dynatrace

1. Log into your Dynatrace tenant.
2. Open the left menu → **Applications & Microservices → Distributed Traces** (or search "Distributed Traces" in the top search bar).
3. You should see live request traffic in the **Requests** timeseries graph, and a table of individual spans showing service name, endpoint, duration, and status — e.g. `Frontend-service`, `ProductCatalog-service`, `adservice`, `cartservice`, `checkoutservice`.
4. Click any row to open the full trace waterfall across services.
5. Use the search box to filter by service name (e.g. `checkoutservice`) to isolate your app's traffic from Kubernetes health checks.

<p align="center">
<img src="image/traces_instrumentation.PNG" width="80%" alt="Dynatrace distributed traces" />
</p>

---

