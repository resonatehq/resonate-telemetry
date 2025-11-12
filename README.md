# Resonate Telemetry

This project provides a self-contained observability sandbox for experimenting with [Resonate](https://github.com/resonatehq/resonate) and [OpenTelemetry](https://opentelemetry.io) metrics and traces using **Prometheus**, **Grafana**, and **Tempo**.

The stack consists of four services:

| Service | Description |
|----------|--------------|
| **Resonate** | Emits prometheus-compatible metrics. |
| **Prometheus** | Collects and stores metrics emitted by Resonate. |
| **Tempo** | Collects distributed traces emitted by Resonate applications via the OpenTelemetry (OTLP) protocol. |
| **Grafana** | Provides a unified dashboard to visualize both metrics and traces. |

## Run

If you already have Resonate running locally:
```
docker-compose up
```

If you do not have Resonate running locally:
```
docker compose --profile with-resonate up -d
```

Once running, the grafana dashboard is available at http://localhost:3000. The default username and password are both `admin`.

When you want to stop the stack, run:
```
docker-compose down
docker compose --profile with-resonate down
```

Resonate applications instrumentalized with OpenTelemetry tracing are available in the traces visualization of the dashboard. Alternatively you can see a representation of your execution by running:
```
resonate tree <id>
```

## Example Resonate application

The following is an example Resonate application instrumentalized with OpenTelemetry tracing. This is not required, if you already running Resonate applications locally metrics and traces (if instrumentalized) should already be available in grafana.

```ts
import { Resonate } from "@resonatehq/sdk";
import { OpenTelemetryTracer } from "@resonatehq/opentelemetry";

import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { NodeSDK } from "@opentelemetry/sdk-node";
import { ATTR_SERVICE_NAME } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  resource: resourceFromAttributes({ [ATTR_SERVICE_NAME]: "resonate" }),
});
sdk.start();

const resonate = new Resonate({
  url: "http://localhost:8001",
  tracer: new OpenTelemetryTracer("resonate"),
});

function* foo(ctx: Context) {
  yield* ctx.run(bar);
}

function* bar(ctx: Context) {
  yield* ctx.run(baz);
}

async function baz(ctx: Context) {
  await new Promise((resolve) => setTimeout(resolve, 1000));

  // simulate failure
  if (Math.random() < 0.5) {
    throw new Error("random failure");
  }
}

resonate.register(foo);
resonate.register(bar);
resonate.register(baz);

const promises = [];

for (let i = 0; i < 100; i++) {
  const h = await resonate.beginRun(crypto.randomUUID(), foo);
  promises.push(h.result())
}

await Promise.all(promises);

resonate.stop();
await sdk.shutdown();
```
