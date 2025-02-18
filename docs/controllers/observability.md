# Observability

This document showcases common techniques for instrumentation:

- **logs** (via [tracing] + [tracing-subscriber] + [EnvFilter])
- **traces** (via [tracing] + [tracing-subscriber] + [opentelemetry-otlp] + [tonic])
- **metrics** (via [tikv/prometheus](https://github.com/tikv/rust-prometheus) exposed via [actix-web])

and follows the approach of [controller-rs].

Most of this logic happens in `main`, before any machinery starts, so it will liberally `.unwrap()`.

## Adding Logs

We will use the [tracing] library for logging because it allows us reusing the same system for tracing later.

```sh
cargo add tracing
cargo add tracing-subscriber --features=json,env-filter
```

We will configure this in `main` by creating a `json` log layer with an [EnvFilter] picking up on the common `RUST_LOG` environment variable:

```rust
let logger = tracing_subscriber::fmt::layer().json();
let env_filter = EnvFilter::try_from_default_env()
    .or_else(|_| EnvFilter::try_new("info"))
    .unwrap();
```

This can be set as the global collector using:

```rust
let collector = Registry::default().with(logger).with(env_filter);
tracing::subscriber::set_global_default(collector).unwrap();
```

We will change how the `collector` is built if using **tracing**, but for now, this is sufficient for adding logging.

## Adding Traces

Following on from logging section, we add extra dependencies to let us push traces to an **opentelemetry** collector (sending over gRPC with [tonic]):

```sh
cargo add opentelemetry --features=trace,rt-tokio
cargo add opentelemetry-otlp --features=tokio
cargo add tonic
```

Setting up the layer and configuring the `collector` follows fundamentally the same process:

```rust
let telemetry = tracing_opentelemetry::layer().with_tracer(init_tracer().await);
```

Note 3 layers now:

```rust
let collector = Registry::default().with(telemetry).with(logger).with(env_filter);
tracing::subscriber::set_global_default(collector).unwrap();
```

However, tracing requires us to have a configurable location of **where to send spans**, so creating the actual `tracer` requires a bit more work:

```rust
async fn init_tracer() -> opentelemetry::sdk::trace::Tracer {
    let otlp_endpoint = std::env::var("OPENTELEMETRY_ENDPOINT_URL")
        .expect("Need a otel tracing collector configured");

    let channel = tonic::transport::Channel::from_shared(otlp_endpoint)
        .unwrap()
        .connect()
        .await
        .unwrap();

    opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_channel(channel))
        .with_trace_config(opentelemetry::sdk::trace::config().with_resource(
            opentelemetry::sdk::Resource::new(vec![opentelemetry::KeyValue::new(
                "service.name",
                "ctlr", // TODO: change to controller name
            )]),
        ))
        .install_batch(opentelemetry::runtime::Tokio)
        .unwrap()
}
```

Note the gRPC address (e.g. `OPENTELEMETRY_ENDPOINT_URL=https://0.0.0.0:55680`) must be explicitly wrapped in a `tonic::Channel`, and this forces an explicit dependency on [tonic].

### Instrumenting

At this point, you can start adding `#[instrument]` attributes onto functions you want, in particular `reconcile`:

```rust
#[instrument(skip(ctx))]
async fn reconcile(foo: Arc<Foo>, ctx: Arc<Data>) -> Result<Action, Error>
```

Note that the `reconcile` span should be **the root span** in the context of a controller. A reconciliation starting is the root of the chain: nothing called into the controller to reconcile an object, this happens regularly automatically.

!!! warning "Higher levels spans"

    Do not `#[instrument]` any function that creates a [Controller] as this would create an unintentionally wide ([application lifecycle wide](https://github.com/kube-rs/kube-rs/pull/741#issuecomment-991163664)) span being a parent to all `reconcile` spans. Such a span will be **problematic** to manage.

### Linking Logs and Traces

To link logs and traces we take advantage that tracing data is being outputted to both logs and our tracing collector, and attach the `trace_id` onto our root span:

```rust
#[instrument(skip(ctx), fields(trace_id))]
async fn reconcile(foo: Arc<Foo>, ctx: Arc<Data>) -> Result<Action, Error> {
    let trace_id = get_trace_id();
    Span::current().record("trace_id", &field::display(&trace_id));
    todo!("reconcile implementation")
}
```

This part is useful for [Loki] or other logging systems as a way to cross-link from logs to traces.

Extracting the `trace_id` requires a helper function atm:

```rust
pub fn get_trace_id() -> opentelemetry::trace::TraceId {
    // opentelemetry::Context -> opentelemetry::trace::Span
    use opentelemetry::trace::TraceContextExt as _;
    // tracing::Span -> opentelemetry::Context
    use tracing_opentelemetry::OpenTelemetrySpanExt as _;

    tracing::Span::current()
        .context()
        .span()
        .span_context()
        .trace_id()
}
```

and it is the only reason for needing to directly add [opentelemetry] as a dependency.

## Adding Metrics

This is the most verbose part of instrumentation because it introduces the need for a [[webserver]], along with data modelling choices and library choices.

We will use [tikv's prometheus library](https://github.com/tikv/rust-prometheus) as its the most battle tested library available:

```sh
cargo add prometheus
```

!!! warning "Limitations"

    The `prometheus` crate outlined herein does not support exemplars nor the openmetrics standard, at current writing. For newer features we will likely look toward the [new official client](https://github.com/prometheus/client_rust), or the [metrics crate suite](https://github.com/metrics-rs/metrics).

### Registering

We will start creating a basic `Metrics` struct to house two metrics, a histogram and a counter:

```rust
#[derive(Clone)]
pub struct Metrics {
    pub reconciliations: IntCounter,
    pub failures: IntCounterVec,
    pub reconcile_duration: HistogramVec,
}

impl Default for Metrics {
    fn default() -> Self {
        let reconcile_duration = HistogramVec::new(
            histogram_opts!(
                "doc_controller_reconcile_duration_seconds",
                "The duration of reconcile to complete in seconds"
            )
            .buckets(vec![0.01, 0.1, 0.25, 0.5, 1., 5., 15., 60.]),
            &[],
        )
        .unwrap();
        let failures = IntCounterVec::new(
            opts!(
                "doc_controller_reconciliation_errors_total",
                "reconciliation errors",
            ),
            &["instance", "error"],
        )
        .unwrap();
        let reconciliations =
            IntCounter::new("doc_controller_reconciliations_total", "reconciliations").unwrap();
        Metrics {
            reconciliations,
            failures,
            reconcile_duration,
        }
    }
}
```

and as these metrics are measurable entirely from within **`reconcile` or `error_policy`** we can attach the struct to the context passed to the [[reconciler##using-context]].

### Measuring

Measuring our metric values can then be done by explicitly taking a `Duration`  inside `reconcile`, but it is easier to wrap this in a struct that relies on `Drop` with a convenience constructor:


```rust
pub struct ReconcileMeasurer {
    start: Instant,
    metric: HistogramVec,
}

impl Drop for ReconcileMeasurer {
    fn drop(&mut self) {
        let duration = self.start.elapsed().as_millis() as f64 / 1000.0;
        self.metric.with_label_values(&[]).observe(duration);
    }
}

impl Metrics {
    pub fn count_and_measure(&self) -> ReconcileMeasurer {
        self.reconciliations.inc();
        ReconcileMeasurer {
            start: Instant::now(),
            metric: self.reconcile_duration.clone(),
        }
    }
}
```

and call this from `reconcile` with one line:

```rust
async fn reconcile(foo: Arc<Foo>, ctx: Arc<Context>) -> Result<Action, Error> {
    let _timer = ctx.metrics.count_and_measure(); // increments now

    // main reconcile body here

    Ok(...) // drop impl invoked, computes time taken
}
```

and handle the `failures` metric inside your  `error_policy`:

```rust
fn error_policy(doc: Arc<Document>, error: &Error, ctx: Arc<Context>) -> Action {
    warn!("reconcile failed: {:?}", error);
    ctx.metrics.reconcile_failure(&doc, error);
    Action::requeue(Duration::from_secs(5 * 60))
}

impl Metrics {
    pub fn reconcile_failure(&self, doc: &Document, e: &Error) {
        self.failures
            .with_label_values(&[doc.name_any().as_ref(), e.metric_label().as_ref()])
            .inc()
    }
}
```

We could increment the failure metric directly, but we have also made a helper function stashed away that extracts the object name and a short error name as labels for the metric.

This type of error extraction requires an impl on your `Error` type. We use `Debug` here:

```rust
impl Error {
    pub fn metric_label(&self) -> String {
        format!("{self:?}").to_lowercase()
    }
}
```

!!! note "Future exemplar work"

    If we had exemplar support, we could have attached our `trace_id` to the histogram metric - through `count_and_measure` - to be able to cross-browse from grafana metric panels into a trace-viewer.

### Exposing

For prometheus to obtain our metrics, we require a web server. As per the [[webserver]] guide, we will assume [actix-web].

In our case, we will pass a `State` struct that contains the `Metrics` struct and attach it to the `HttpServer` in `main`:

```rust
HttpServer::new(move || {
    App::new()
        .app_data(Data::new(state.clone())) // new state
        .service(metrics) // new endpoint
    })
```

the `metrics` service is the important one here, and its implementation is able to extract the `Metrics` struct from actix's `web::Data`:

```rust
#[get("/metrics")]
async fn metrics(c: web::Data<State>, _req: HttpRequest) -> impl Responder {
    let metrics = c.metrics(); // grab out of actix data
    let encoder = TextEncoder::new();
    let mut buffer = vec![];
    encoder.encode(&metrics, &mut buffer).unwrap();
    HttpResponse::Ok().body(buffer)
}
```

### What Metrics

The included metrics `failures`, `reconciliations` and a `reconcile_duration` histogram will be sufficient to have prometheus compute a wide array of details:

- reconcile amounts in last hour - `sum(increase(reconciliations[1h]))`
- hourly error rates - `sum(rate(failures[1h]) / sum(rate(reconciliations[1h]))`
- success rates - same rate setup but `reconciliations / (reconciliations + failures)`
- p90 reconcile duration - `histogram_quantile(0.9, sum(rate(reconciliations[1h])))`

and you could then create alerts on aberrant values (e.g. say 10% error rate, zero reconciliation rate, and maybe p90 durations >30s).

The above metric setup should comprise the core need of a **standard** controller (although you may have [more things to care about](https://sirupsen.com/metrics) than our simple example).

!!! note "kube-state-metrics"

    It is possible to derive metrics from conditions and fields in your CRD schema using [runtime flags to `kube-state-metrics`](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/customresourcestate-metrics.md) without instrumentation, but since this is an implicit dependency for operators, it should not be a default.

You will also want resource utilization metrics, but this is typically handled upstream. E.g. cpu/memory utilization metrics are generally available via kubelet's metrics and other utilization metrics can be gathered from [node_exporter](https://github.com/prometheus/node_exporter).

!!! note "tokio-metrics"

    New **experimental** runtime metrics are also availble for the tokio runtime via [tokio-metrics](https://github.com/tokio-rs/tokio-metrics).

### External References

- [Metrics in axum using metrics crate](https://github.com/tokio-rs/axum/tree/143c415955bbd5021e28f493ef7c285de191ffe1/examples/prometheus-metrics)



--8<-- "includes/abbreviations.md"
--8<-- "includes/links.md"

[//begin]: # "Autogenerated link references for markdown compatibility"
[webserver]: webserver "Web Server"
[reconciler##using-context]: reconciler "The Reconciler"
[//end]: # "Autogenerated link references"
