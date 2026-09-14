# OpenTelemetry
openTelemetry-without-OneAgent

**Flow from Code:**

App code (OTel SDK) → generates spans → OTel Collector (receives, batches) → exports via OTLP → Dynatrace ingest → Distributed Traces UI

<img width="1703" height="927" alt="image" src="https://github.com/user-attachments/assets/e593d87a-bfb8-429c-b841-e84687b99915" />


