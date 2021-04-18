---
layout: post
title: 'OpenTelemetry: Observability in Distributed Services'
date: '2021-04-06T09:08:00.000-07:00'
categories: git
author: dReXler
tags:
- opentelemetry 
- rust
---

#### Prologue
In the past year, the platform team was tasked with setting up infrastructure and services to unify the company's collection of disparate databases into distinct domain databases. These legacy databases served a wide array of business applications and ran on different RDBMSes. The overall objective was to eventually have a set of microservices each of which encapsulated a business concern, upon which teams could then build their applications and services. An interesting project. The challenge? It had to be done gradually without disruption to other teams' applications. Until those legacy databases were retired, changes to any domain database needed to be streamed in realtime to them. Briefly, based on the initial requirements, we went with PostgreSQL to power the domain databases; Debezium to capture data changes in those databases' Write Ahead Logs and forward them for targeted schema conversions with dedicated services via Kinesis.

After a couple of development iterations, to gain an insight into the performance of the service that polled the WAL, the service wrapper around Debezium was instrumented using [Micrometer] with the generated metrics scraped by an external Prometheus cluster. Subsequently, a Grafana dashboard was setup to visualize these.  With the removal of a key requirement of maintaining the order of data changes transmitted downstream, the need for reading the WAL also changed as domain databases now each had a dedicated outbox to commit transaction summaries to. Naturally, the question became whether Debezium would still be necessary. It was not. The team would go on to prove that out by building a replacement containerized multithreaded service that polled the *Outbox* table and sharded the changes by tenant into Kinesis. A couple of interesting things of note happened here:
* *Implementation was in .Net Core*
* *The instrumentation library changed to [AppMetrics]*
* *The metrics dashboard was updated to visualize the newly generated data*

That's quite a few changes. The curious Rustacean in me wondered how the performance of this service would compare if it was written in [Rust] to take advantage of the concurrent primitives and performance [Tokio] provides alongside [Rusoto]. Replicating it would be fairly straightforward, however, there was no avoiding the other two changes: a new language-specific metrics library and updating the dashboards all over again. How can all this be avoided, not just in this scenario but in an much wider context of rewrites of existing services but needing to preserve the existing instrumentation? Additionally, internal deliberations about the possibility of moving away from the APM vendor to an internally hosted Elastic solution was also another real factor for consideration. Incredibly, a new open source project [OpenTelemetry] had a lot of the answers to questions I was mulling.

#### OpenTelemetry
[OpenTelemetry] is a [CNCF] project that defines a language-neutral specification and provides a collection of APIs, SDKs for handling observability data such as logs, metrics & traces in a vendor-agnostic manner. This project was formed from the convergence of two competing projects- OpenTracing & OpenCensus and backed by major cloud providers from Google, Microsoft, Amazon and virtually all vendors in the observability space - Splunk, Elastic, Datadog, LightStep, DynaTrace, NewRelic, Logzio, HoneyComb etc. Let us explore the benefits of adopting OpenTelemetry for existing and future greenfield projects.

* The [OpenTelemetry Specification]'s language neutrality allows for implementations in different languages. Currently, as of this writing, there are implementations for some of the most widely used general purpose languages provided by OpenTelemetry's SIGs -  Special Interest Groups: C++, .Net, Java, Javascript, Python, Go, Rust, Erlang, Ruby, PHP, Swift. These are a dedicated groups of contributors with a focus on a single language implementation. If there's a software project using a language that is unsupported currently, chances are it will be supported in future. All this means a greater degree of flexibility when implementing software components; regardless of the language choice, instrumentation will be the same.
  
* [OpenTelemetry]'s extensible architecture means that library/plugin authors can instrument their code with the API and when these artifacts are utilized in a service or application implementing the OpenTelemetry SDK, there is visibility into both the service code and third party libraries performance. Microsoft's [Distributed Application Runtime] library is an example. There are plugins for popular frameworks like Spring, Express etc.
  
* [OpenTelemetry] prevents vendor lock-in. The OpenTelemetry [Collector] allows for the reception, processing and exportation of telemetry data with support for different open source wire formats - Jaeger, Prometheus, Fluent Bit, W3C TraceContext format, Zipkin's B3 headers etc. More so, with implementation of [Exporters] for different telemetry backends, switching between vendors is a breeze. For example, one can pipe their tracing data to NewRelic, Elastic, a deployed instance of Zipkin etc...and it is all a simple configuration change on the [Collector]. Think of it as instrumentation as a form of abstraction, where the destination backends for the telemetry data is abstracted away from the application/service.
  
  ![OpenTelemetry Collector](/assets/imgs/otel-collector.png)

* With the stabilization of the [Tracing Specification] and the outline of the [Metrics Roadmap], OpenTelemetry is shaping up to be the way to derive insights into distributed services. One library to trace, generate metrics and connect them to other telemetry data. Since it is also the [CNCF] project to replace OpenTracing & OpenCensus, for service meshes like [Linkerd], [OpenTelemetry] will eventually become the de-facto way of propagating telemetry data from the various services. This means an easier transition when moving from a collection of microservices to a service mesh when the complexity warrants it.

#### Demo

 ![OpenTelemetry Collector](/assets/imgs/otel-demo-tracing.png)


For a quick demonstration of the tracing capabilities, I have a [demo] built to showcase:
* support for multiple languages -  chose Rust, Typescript and .Net 5
* tracing across different communication protocols - HTTP & gRPC
* auto-instrumentation of services
* manual instrumentation of services

Rust is intentionally used for two services - *Employee* & *Direct Deposits* to demonstrate manual instrumentation with synchronous and asynchronous functions as the data layer each service works with offered a sync API, in the case of [Diesel] with PostgreSQL and an async API via [MongoDB]'s Rust 2.0-alpha driver. Typescript, comes along for the ride since it's one of the languages I use server side. .Net5 is latest language iteration from Microsoft, so I had a side interest in taking a peek at its comparative performance to Rust [again]. 

##### Instrumentation, Traces, Events & Tags
With OpenTelemetry, one can auto-instrument code and/or apply manual instrumentation. This gives flexibility when working with legacy codebases or starting greenfield projects. 

From the demo code, I the instrumentation with .Net5-based *Paycheck* service as follows:
{% highlight csharp %}
{% github_sample /drexler/opentelemetry-demo-tracing/blob/main/services/paycheck/src/DependencyInjection.cs 67 83 %}
{% endhighlight %}

Here, under the hood, OpenTelemetry for DotNet set up the auto-instrumentation via *AddAspNetCoreInstrumentation()* and uses the *System.Diagnostics.ActivitySource* to setup the event sink named *paycheck-db-conn* for handling the manual instrumentation later as seen below: 

{% highlight csharp %}
{% github_sample /drexler/opentelemetry-demo-tracing/blob/main/services/paycheck/src/Repositories/PayRepository.cs 12 43 %}
{% endhighlight %}

A trace is simply a collection of spans. In the above, we start a *child* span of the parent span and annotate it appropriate via tags & events. Events represent occurences that happened at a specific time during a span's workload. Together, the additional metadata drives quick insights when investigating problems. For example suppose we got the following api response on an attempt to load all paychecks:

{% highlight json %}
{
    "statusCode": 500,
    "message": "Server ist kaput!",
    "developerMessage": "Internal Error",
    "requestId": "451b8025562676951540a00cc121af04"
}
{% endhighlight %}

 With the requestId (aka the traceId) above, we see how the additional metadata which we applied to the span helps us to understand a request error. 

 ![Trace Overview](/assets/imgs/trace-error.png) 

 ![Database Trace Details](/assets/imgs/db-down.png) 
 
 In this case, a database connectivity problem! 

 We see this same pattern applied to both the Rust-based services: *Employee* & *Direct Deposit*
{% highlight rust %}
{% github_sample /drexler/opentelemetry-demo-tracing/blob/main/services/employee/src/database.rs 23 48 %}
{% endhighlight %} 

and the *Payroll* service: 
{% highlight typescript %}
{% github_sample /drexler/opentelemetry-demo-tracing/blob/main/services/payroll/src/routes/employees.ts 89 111 %}
{% endhighlight %} 


##### Collector Deployment, Pipelines & Telemetry-backend Configuration

There are two modes of deploying the OpenTelemetry Collector: 
* agent: where the collector instance is running on the same host as client application
* gateway: where one or more collector instances run as a standalone service.

The Collector serves as centralized point to allow for the configuration of telemetry data reception, processing and exportation to desired telemetry backends. In our demo configuration as shown below, we pushed trace data in *batches* that was *received* via HTTP & gRPC, through a trace *pipeline* setup to *export* the data to the [Jaeger] and [Zipkin] instances we deployed for analysis. Although commented out, the traces could have been sent to NewRelic as well. There are many supported vendors with the Collector. This makes it easy to compare the depth of analytics that various vendors can provide given the same telemetry data.

{% highlight yaml %}
{% github_sample /drexler/opentelemetry-demo-tracing/blob/main/collector-config.yaml 0 29 %}
{% endhighlight %}  

Through pipelines, metrics and eventually logs data will be exported through the collector thus making OpenTelemetry the one library for applications' observability needs. Additionally, with the Collector pipelines, one can ship various telemetry data to different vendors. If desired, one can ship their metrics data to a vendor like Splunk and still use DataDog for traces and logging. OpenTelemetry opens up the possibilities...


##### Links
The demo code for the above can be found [here](https://github.com/drexler/opentelemetry-demo-tracing)



[OpenTelemetry]: https://opentelemetry.io/
[Rust]: https://www.rust-lang.org/
[Micrometer]: https://micrometer.io/
[AppMetrics]: https://www.app-metrics.io/
[Tokio]: https://tokio.rs/
[CNCF]: https://www.cncf.io/
[Collector]: https://opentelemetry.io/docs/collector/
[Tracing Specification]: https://medium.com/opentelemetry/opentelemetry-specification-v1-0-0-tracing-edition-72dd08936978
[Metrics Roadmap]: https://medium.com/opentelemetry/opentelemetry-metrics-roadmap-f4276fd070cf
[Linkerd]: https://linkerd.io/
[Rusoto]: https://github.com/rusoto/rusoto
[OpenTelemetry Specification]: https://github.com/open-telemetry/opentelemetry-specification
[Exporters]: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter
[Distributed Application Runtime]: https://dapr.io/
[Diesel]: http://diesel.rs/
[MongoDB]: https://www.mongodb.com/2
[again]: https://drexler.github.io/aws-lambda-rust/
[Jaeger]: https://www.jaegertracing.io/
[Zipkin]: https://zipkin.io/
[demo]: https://github.com/drexler/opentelemetry-demo-tracing 