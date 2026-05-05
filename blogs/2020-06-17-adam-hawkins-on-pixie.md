---
title: "Adam Hawkins On Pixie"
url: "https://blog.px.dev/adam-hawkins-on-pixie/"
date: "Wed, 17 Jun 2020 00:00:00 GMT"
author: ""
feed_url: "https://blog.px.dev/rss.xml"
---
<p>I, <a href="https://hawkins.io">Adam Hawkins</a>, recently tried Pixie. I was
instantly impressed because it solved a recurring problem for me:
application code changes. Let me explain.</p><p>As an SRE, I&#x27;m responsible for operations, but am often unaware of the
services internals. These services
are black boxes to me. If the box is an HTTP service, then that
requires telemetry on incoming request counts, latencies, and
response status code--bonus points for p50, p90, and p95 latencies. My
problem, and I&#x27;m guessing it&#x27;s common to other SRE and DevOps teams,
is that these services are often improperly instrumented. Before
Pixie, we would have to wait on the dev team to add the required
telemetry. Truthfully, that&#x27;s just toil. It would be better for
SREs, DevOps engineers, and application developers to have
telemetry provided automatically via infrastructure. Enter Pixie.</p><p>Pixie is the first telemetry tool I&#x27;ve seen that provides
operational telemetry out of the box with <strong>zero</strong> changes to
application code. SREs can simply run <code>px deploy</code>, start collecting
data, then begin troubleshooting in minutes.</p><p>It took me a bit to grok Pixie because it&#x27;s different than
tools like NewRelic or DataDog that I&#x27;ve used in the past. Tools like
these are different than Pixie becauase:</p><ul><li>They require application code changes (like adding in
client library or annotating Kubernetes manifests) to gather full
telemetry.</li><li>They&#x27;re largely GUI driven.</li><li>Telemetry is collected then shipped off to a centralized service
(which drives up the cost).</li></ul><p>Pixie is radically different.</p><ul><li>First, it integrates with eBPF so it can
collect data about application traffic without application code
changes.  Pixie provides common HTTP telemetry (think request counts,
latencies, and status codes) for all services running on your
Kubernetes cluster.  Better yet, Pixie generates service to service
telemetry, so you&#x27;re given a service map right out of the box.</li><li>Second, it bakes infrastructure-as-code principles into the core DX. Every
Pixie Dashboard is a program, which you can manage with version
control and distribute amongst your team, or even take with to
different teams. Pixie also provides a terminal interface so you can
interact with the dashboards directly in the CLI. That&#x27;s a first for
me and I loved it! These same scripts can also run in the browser.</li><li>Third, and lastly, Pixie&#x27;s data storage and pricing model is
different. Pixie keeps all telemetry data on your cluster, as a result
the cost is signicantly lower. It&#x27;s easy to pay $XXX,XXX dollars per
year for other tools. Pixie&#x27;s cost promises to be orders of
magnitude less.</li></ul><p>Sounds interesting right?</p><p>Check out my demo video for quick
walkthrough.</p><p>Pixie is in free community beta right now. You can install it on your
cluster and try it for yourself.</p>
