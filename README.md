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


