---
title: "A case of Datadog APM not stitching traces"
date: 2022-02-17T22:16:50-08:00
draft: false
---

Recently I worked on instrumenting a service with Datadog APM (Application Performance Monitoring). It's when I ran into an interesting issue that the StackOverflow didn't offer the usual obvious answer. After some digging, I'm finally sharing my solution it here.

The problem was that sometimes traces produced by two services didn't get glued together by the Datadog backend. It appeared that APM traces with calls between "Service A" and "Service B" were correlated, but the same stitching didn't work when "Service B" made a call to "Service A".

After some digging to find out what was wrong with my Datadog agent configuration, I realized that I needed to take a step back and ask "How does Datadog correlate traces?".

In order for Datadog to stitch together two traces received from two different tracers (in my case - two agents running in two different services), the trace ID/parent trace ID have to match. 

Datadog automatically injects the trace ID but it's not very well documented what is involved. At least not in Datadog's documentation.

Finally I found a useful page: [how are traces linked to tests](https://docs.datadoghq.com/synthetics/apm/#how-are-traces-linked-to-tests). Based on this article, I learned that the Datadog agent can automatically inject headers in outgoing HTTP calls. 

I looked at the [Java tracer source code](https://github.com/DataDog/dd-trace-java/blob/v0.95.1/dd-java-agent/instrumentation/opentelemetry/src/main/java/datadog/trace/instrumentation/opentelemetry/OtelContextPropagators.java) and learned that it actually follows [OpenTelemetry](https://opentelemetry.io/docs/) convention for [context propagation](https://opentelemetry.lightstep.com/core-concepts/context-propagation/). The tracer agent, running within the same process as my Java application, intercepts any HTTP requests and sprinkles some extra headers to help corrlate the spans.

Why was the context propagation broken? Knowing that I should expect extra headers to show up in the calls made between the services, I was able to confirm that the headers were simply missing. In my particular case, this was due to the way nginx reverse proxy was configured for the "Service A". The nginx.conf contained the following line:
```
proxy_pass_request_headers off;
```
It means that the headers would not be passed to the upstream server unless explicitly requested with `proxy_set_header`. Therefore, the headers were missing by design.

From that the fix was easy. I added the following lines to nginx.conf:
```
location / {
    # ...

    proxy_set_header x-datadog-trace-id $http_x_datadog_trace_id;
    proxy_set_header x-datadog-parent-id $http_x_datadog_parent_id;
    proxy_set_header x-datadog-sampling-priority $http_x_datadog_sampling_priority;
    proxy_set_header
    x-datadog-origin $http_x_datadog_origin;
    
    # ... 
}
```

And that worked! Traces are now auto-magically stitched together. Hope this helps someone solve their configuration problem a bit quicker.

