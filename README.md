# Resonate Telemetry

## Run
```
docker-compose up
```

The grafana dashboard is available at http://localhost:3000. The default username and password are both `admin`.

## Instrumentalize application
```ts
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { NodeSDK } from "@opentelemetry/sdk-node";
import { ATTR_SERVICE_NAME } from "@opentelemetry/semantic-conventions";
import { OpenTelemetryTracer } from "@resonatehq/opentelemetry";
import { Resonate } from "@resonatehq/sdk";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  resource: resourceFromAttributes({ [ATTR_SERVICE_NAME]: "resonate" }),
  instrumentations: [ getNodeAutoInstrumentations() ],
});
sdk.start();

const resonate = new Resonate({
  url: "http://localhost:8001",
  tracer: new OpenTelemetryTracer("resonate"),
});

// ...

resonate.stop();
await sdk.shutdown();
```
