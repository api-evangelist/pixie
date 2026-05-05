---
title: "Horizontal Pod Autoscaling with Custom Metrics in Kubernetes"
url: "https://blog.px.dev/autoscaling-custom-k8s-metric/"
date: "Wed, 20 Oct 2021 00:00:00 GMT"
author: ""
feed_url: "https://blog.px.dev/rss.xml"
---
<p><strong>Sizing a Kubernetes deployment can be tricky</strong>. How many pods does this deployment need? How much CPU/memory should I allocate per pod? The optimal number of pods varies over time, too, as the amount of traffic to your application changes.</p><p>In this post, we&#x27;ll walk through how to autoscale your Kubernetes deployment by custom application metrics. As an example, we&#x27;ll use Pixie to generate a custom metric in Kubernetes for HTTP requests/second by pod.</p><p>Pixie is a fully open source, CNCF sandbox project that can be used to generate a wide range of custom metrics. However, the approach detailed below can be used for any metrics datasource you have, not just Pixie.</p><p>The full source code for this example lives <a href="https://github.com/pixie-io/pixie-demos/tree/main/custom-k8s-metrics-demo">here</a>. If you want to go straight to autoscaling your deployments by HTTP throughput, it can be used right out of the box.</p><h2>Metrics for autoscaling</h2><p>Autoscaling allows us to automatically allocate more pods/resources when the application is under heavy load, and deallocate them when the load falls again. This helps to provide a stable level of performance in the system without wasting resources.</p><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p><strong>The best metric to select for autoscaling depends on the application</strong>. Here is a (very incomplete) list of metrics that might be useful, depending on the context:</p><ul><li>CPU</li><li>Memory</li><li>Request throughput (HTTP, SQL, Kafka…)</li><li>Average, P90, or P99 request latency</li><li>Latency of downstream dependencies</li><li>Number of outbound connections</li><li>Latency of a specific function</li><li>Queue depth</li><li>Number of locks held</li></ul><p>Identifying and generating the right metric for your deployment isn&#x27;t always easy. CPU or memory are tried and true metrics with wide applicability. They&#x27;re also comparatively easier to grab than application-specific metrics (such as request throughput or latency).</p><p><strong>Capturing application-specific metrics can be a real hassle.</strong> It&#x27;s a lot easier to fall back to something like CPU usage, even if it doesn&#x27;t reflect the most relevant bottleneck in our application. In practice, just getting access to the right data is half the battle. Pixie can automatically collect telemetry data with <a href="https://docs.px.dev/about-pixie/pixie-ebpf/">eBPF</a> (and therefore without changes to the target application), which makes this part easier.</p><p>The other half of the battle (applying that data to the task of autoscaling) is well supported in Kubernetes!</p><h2>Autoscaling in Kubernetes</h2><p>Let&#x27;s talk more about the options for autoscaling deployments in Kubernetes. Kubernetes offers two types of autoscaling for pods.</p><p><strong>Horizontal Pod Autoscaling</strong> (<a href="https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/">HPA</a>) automatically increases/decreases the <em>number</em> of pods in a deployment.</p><p><strong>Vertical Pod Autoscaling</strong> (<a href="https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler">VPA</a>) automatically increases/decreases <em>resources</em> allocated to the pods in your deployment.</p><p>Kubernetes provides built-in support for autoscaling deployments based on resource utilization. Specifically, you can autoscale your deployments by CPU or Memory with just a few lines of YAML:</p><pre><code class="language-yaml">apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-cpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
</code></pre><p>This makes sense, because CPU and memory are two of the most common metrics to use for autoscaling. However, like most of Kubernetes, Kubernetes autoscaling is also <em>extensible</em>. Using the Kubernetes custom metrics API, <strong>you can create autoscalers that use custom metrics that you define</strong> (more on this soon).</p><p>If I&#x27;ve defined a custom metric, <code>my-custom-metric</code>, the YAML for the autoscaler might look like this:</p><pre><code class="language-yaml">apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: my-custom-metric
      target:
        type: AverageValue
        averageUtilization: 20
</code></pre><p>How can I give the Kubernetes autoscaler access to this custom metric? We will need to implement a custom metric API server, which is covered next.</p><h2>Kubernetes Custom Metric API</h2><p>In order to autoscale deployments based on custom metrics, we have to provide Kubernetes with the ability to fetch those custom metrics from within the cluster. This is exposed to Kubernetes as an API, which you can read more about <a href="https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md">here</a>.</p><p>The custom metrics API in Kubernetes associates each metric with a particular resource:</p><p><code>/namespaces/example-ns/pods/example-pod/{my-custom-metric}</code>
fetches <code>my-custom-metric</code> for pod <code>example-ns/example-pod</code>. </p><p>The Kubernetes custom metrics API also allows fetching metrics by selector:</p><p><code>/namespaces/example-ns/pods/*/{my-custom-metric}</code>
fetches <code>my-custom-metric</code> for all of the pods in the namespace <code>example-ns</code>. </p><p><strong>In order for Kubernetes to access our custom metric, we need to create a custom metric server that is responsible for serving up the metric.</strong> Luckily, the <a href="https://github.com/kubernetes/community/tree/master/sig-instrumentation">Kubernetes Instrumentation SIG</a> created a <a href="https://github.com/kubernetes-sigs/custom-metrics-apiserver">framework</a> to make it easy to build custom metrics servers for Kubernetes.</p><div class="image-l"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>All that we needed to do was implement a Go server meeting the framework&#x27;s interface:</p><pre><code class="language-go">type CustomMetricsProvider interface {
    // Fetches a single metric for a single resource.
    GetMetricByName(ctx context.Context, name types.NamespacedName, info CustomMetricInfo, metricSelector labels.Selector) (*custom_metrics.MetricValue, error)

    // Fetch all metrics matching the input selector, i.e. all pods in a particular namespace.
    GetMetricBySelector(ctx context.Context, namespace string, selector labels.Selector, info CustomMetricInfo, metricSelector labels.Selector) (*custom_metrics.MetricValueList, error)

    // List all available metrics.
    ListAllMetrics() []CustomMetricInfo
}
</code></pre><h2>Implementing a Custom Metric Server</h2><p>Our implementation of the custom metric server can be found <a href="https://github.com/pixie-io/pixie-demos/blob/main/custom-k8s-metrics-demo/pixie-http-metric-provider.go">here</a>. Here&#x27;s a high-level summary of the basic approach:</p><ul><li>In <code>ListAllMetrics</code>, the custom metric server defines a custom metric, <code>px-http-requests-per-second</code>, on the Pod resource type.</li><li>The custom metric server queries Pixie&#x27;s <a href="https://docs.px.dev/reference/api/overview">API</a> every 30 seconds in order to generate the metric values (HTTP requests/second, by pod).</li><li>These values can be fetched by subsequent calls to <code>GetMetricByName</code> and <code>GetMetricBySelector</code>.</li><li>The server caches the results of the query to avoid unnecessary recomputation every time a metric is fetched.</li></ul><p>The custom metrics server contains a <a href="https://github.com/pixie-io/pixie-demos/blob/main/custom-k8s-metrics-demo/pixie-http-metric-provider.go#L33-L48">hard-coded</a> PxL script (Pixie&#x27;s <a href="https://docs.px.dev/reference/pxl/">query language</a>) in order to compute HTTP requests/second by pod. PxL is very flexible, so we could easily extend this script to generate other metrics instead (latency by pod, requests/second in a different protocol like SQL, function latency, etc). </p><p>It&#x27;s important to generate a custom metric for every one of your pods, because the Kubernetes autoscaler will not assume a zero-value for a pod without a metric associated. One early bug our implementation had was omitting the metric for pods that didn&#x27;t have any HTTP requests.</p><h2>Testing and Tuning</h2><p>We can sanity check our custom metric via the <code>kubectl</code> API:</p><pre><code class="language-bash">kubectl [-n &lt;ns&gt;] get --raw &quot;/apis/custom.metrics.k8s.io/v1beta2/namespaces/default/pods/*/px-http-requests-per-second&quot;
</code></pre><p>Let&#x27;s try it on a demo application, a simple echo server. The echo server has a horizontal pod autoscaler that looks like this:</p><pre><code class="language-yaml">apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: echo-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: echo-service
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: px-http-requests-per-second
        target:
          type: AverageValue
          averageValue: 20
</code></pre><p><strong>This autoscaler will try to meet the following goal: 20 requests per second (on average), per pod.</strong> If there are more requests per second, it will increase the number of pods, and if there are fewer, it will decrease the number of pods. The <code>maxReplicas</code> property prevents the autoscaler from provisioning more than 10 pods.</p><p>We can use <a href="https://github.com/rakyll/hey">hey</a> to generate artificial HTTP load. This generates 10,000 requests to the <code>/ping</code> endpoint in our server.</p><pre><code class="language-bash">hey -n 10000 http://&lt;custom-metric-server-external-ip&gt;/ping
</code></pre><p>Let&#x27;s see what happens to the number of pods over time in our service. In the chart below, the 10,000 requests were generated at second 0. The requests were fast -- all 10,000 completed within a few seconds.</p><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>This is pretty cool. While all of the requests completed within 5 seconds, it took about 100 seconds for the autoscaler to max out the number of pods.</p><p>This is because <strong>Kubernetes autoscaling has intentional delays by default</strong>, so that the system doesn&#x27;t scale up or down too quickly due to spurious load. However, we can configure the <code>scaleDown</code>/<code>scaleUp</code> behavior to make it add or remove pods more quickly if we want:</p><pre><code class="language-yaml">apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: echo-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: echo-service
  minReplicas: 1
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 0
    scaleUp:
      stabilizationWindowSeconds: 0
  metrics:
    - type: Pods
      pods:
        metric:
          name: px-http-requests-per-second
        target:
          type: AverageValue
          averageValue: 20
</code></pre><p>This should increase the responsiveness of the autoscaler. Note: there are other parameters that you can configure in <code>behavior</code>, some of which are listed <a href="https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-configurable-scaling-behavior">here</a>.</p><p>Let&#x27;s compare the new autoscaling configuration to the old one applied with the exact same input stimulus: 10,000 pings from <code>hey</code> at second 0.</p><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>We can see that in comparison to the original configuration, the second version of our autoscaler adds and removes pods more quickly, as intended. <strong>The optimal values for these parameters are highly context-dependent</strong>, so you’ll want to consider the pros and cons of faster or slower stabilization for your own use case.</p><h2>Final Notes and Extensibility</h2><p>As mentioned, you can check out the full source code for the example <a href="https://github.com/pixie-io/pixie-demos/tree/main/custom-k8s-metrics-demo">here</a>. Instructions for out-of-the-box use of the HTTP load balancer are also included. You can also tweak the script in the example to produce a different metric, such as HTTP request latency, or perhaps throughput of a different request <a href="https://docs.px.dev/about-pixie/data-sources/#supported-protocols">protocol</a>.</p><p>Thanks again to the Kubernetes Instrumentation SIG for making the framework for the custom metrics server!</p><p>Questions? Find us on <a href="https://slackin.px.dev/">Slack</a> or Twitter at <a href="https://twitter.com/pixie_run">@pixie_run</a>.</p>
