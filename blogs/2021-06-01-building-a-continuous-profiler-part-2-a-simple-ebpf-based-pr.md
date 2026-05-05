---
title: "Building a Continuous Profiler Part 2: A Simple eBPF-Based Profiler"
url: "https://blog.px.dev/cpu-profiling-2/"
date: "Tue, 01 Jun 2021 00:00:00 GMT"
author: ""
feed_url: "https://blog.px.dev/rss.xml"
---
<p>In the last <a href="/cpu-profiling/#part-1:-an-introduction-to-application-performance-profiling">blog post</a>, we discussed the basics of CPU profilers for compiled languages like Go, C++ and Rust. We ended by saying we wanted a sampling-based profiler that met these two requirements:</p><ol><li><p>Does not require recompilation or redeployment: This is critical to Pixie’s auto-telemetry approach to observability. You shouldn’t have to instrument or even re-run your application to get observability.</p></li><li><p>Has very low overheads: This is required for a continuous (always-on) profiler, which was desirable for making performance profiling as low-effort as possible.</p></li></ol><p>A few existing profilers met these requirements, including the Linux <a href="https://github.com/torvalds/linux/tree/master/tools/perf">perf</a> tool. In the end, we settled on the BCC eBPF-based profiler developed by Brendan Gregg <a href="http://www.brendangregg.com/blog/2016-10-21/linux-efficient-profiler.html">[1]</a> as the best reference. With eBPF already at the heart of the Pixie platform, it was a natural fit, and the efficiency of eBPF is undeniable.</p><p>If you’re familiar with eBPF, it’s worth checking out the source code of the <a href="https://github.com/iovisor/bcc/blob/v0.20.0/tools/profile.py">BCC implementation</a>. For this blog, we’ve prepared our own simplified version that we’ll examine in more detail.</p><h2>An eBPF-based profiler</h2><p>The code to our simple eBPF-based profiler can be found <a href="https://github.com/pixie-io/pixie-demos/tree/main/ebpf-profiler">here</a>, with further instructions included at the end of this blog (see <a href="/cpu-profiling-2/#running-the-demo-profiler">Running the Demo Profiler</a>). We’ll be explaining how it works, so now’s a good time to clone the repo.</p><p>Also, before diving into the code, we should mention that the Linux developers have already put in dedicated hooks for collecting stack traces in the kernel. These are the main APIs we use to collect stack traces (and this is how the official BCC profiler works well). We won’t, however, go into Linux’s implementation of these APIs, as that’s beyond the scope of this blog.</p><p>With that said, let’s look at some BCC eBPF code. Our basic structure has three main components:</p><pre><code class="language-cpp">const int kNumMapEntries = 65536;

BPF_STACK_TRACE(stack_traces, kNumMapEntries);

BPF_HASH(histogram, struct stack_trace_key_t, uint64_t, kNumMapEntries);

int sample_stack_trace(struct bpf_perf_event_data* ctx) {
 // Collect stack traces
 // ...
}
</code></pre><p>Here we define:</p><ol><li><p>A <code>BPF_STACK_TRACE</code> data structure called <code>stack_traces</code> to hold sampled stack traces. Each entry is a list of addresses representing a stack trace. The stack trace is accessed via an assigned stack trace ID.</p></li><li><p>A <code>BPF_HASH</code> data structure called <code>histogram</code> which is a map from the sampled location in the code to the number of times we sampled that location.</p></li><li><p>A function <code>sample_stack_trace</code> that will be periodically triggered. The purpose of this eBPF function is to grab the current stack trace whenever it is called, and to populate/update the <code>stack_traces</code> and <code>histogram</code> data structures appropriately.</p></li></ol><p>The diagram below shows an example organization of the two data structures.</p><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>As we’ll see in more detail later, we’ll set up our BPF code to trigger on a periodic timer. This means every X milliseconds, we’ll interrupt the CPU and trigger the eBPF probe to sample the stack traces. Note that this happens regardless of which process is on the CPU, and so the eBPF profiler is actually a system-wide profiler. We can later filter the results to include only the stack traces that belong to our application.</p><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>Now let’s look at the full BPF code inside <code>sample_stack_trace</code>:</p><pre><code class="language-cpp">int sample_stack_trace(struct bpf_perf_event_data* ctx) {
 // Sample the user stack trace, and record in the stack_traces structure.
 int user_stack_id = stack_traces.get_stackid(&amp;ctx-&gt;regs, BPF_F_USER_STACK);

 // Sample the kernel stack trace, and record in the stack_traces structure.
 int kernel_stack_id = stack_traces.get_stackid(&amp;ctx-&gt;regs, 0);

 // Update the counters for this user+kernel stack trace pair.
 struct stack_trace_key_t key = {};
 key.pid = bpf_get_current_pid_tgid() &gt;&gt; 32;
 key.user_stack_id = user_stack_id;
 key.kernel_stack_id = kernel_stack_id;
 histogram.increment(key);

 return 0;
}
</code></pre><p>Surprisingly, that’s it! That’s the entirety of our BPF code for our profiler. Let’s break it down...</p><p>Remember that an eBPF probe runs in the context when it was triggered, so when this probe gets triggered it has the context of whatever program was running on the CPU. Then it essentially makes two calls to <code>stack_traces.get_stackid()</code>: one to get the current user-code stack trace, and another to get the kernel stack trace. If the code was not in kernel space when interrupted, the second call simply returns EEXIST, and there is no stack trace. You can see that all the heavy-lifting is really done by the Linux kernel.</p><p>Next, we want to update the counts for how many times we’ve been at this exact spot in the code. For this, we simply increment the counter for the entry in our histogram associated with the tuple {pid, user_stack_id, kernel_stack_id}. Note that we throw the PID into the histogram key as well, since that will later help us know which process the stack trace belongs to.</p><h2>We’re Not Done Yet</h2><p>While the eBPF code above samples the stack traces we want, we still have a little more work to do. The remaining tasks involve:</p><ol><li><p>Setting up the trigger condition for our BPF program, so it runs periodically.</p></li><li><p>Extracting the collecting data from BPF maps.</p></li><li><p>Converting the addresses in the stack traces into human readable symbols.</p></li></ol><p>Fortunately, all this work can be done in user-space. No more eBPF required.</p><p>Setting up our BPF program to run periodically turns out to be fairly easy. Again, credit goes to the BCC and eBPF developers. The crux of this setup is the following:</p><pre><code class="language-cpp">bcc-&gt;attach_perf_event(
  PERF_TYPE_SOFTWARE,
  PERF_COUNT_SW_CPU_CLOCK,
  std::string(probe_fn),
  sampling_period_millis * kNanosPerMilli,
  0);
</code></pre><p>Here we’re telling the BCC to set up a trigger based on the CPU clock by setting up an event based on <code>PERF_TYPE_SOFTWARE/PERF_COUNT_SW_CPU_CLOCK</code>. Every time this value reaches a multiple of <code>sampling_period_millis</code>, the BPF probe will trigger and call the specified <code>probe_fn</code>, which happens to be our <code>sample_stack_trace</code> BPF program. In our demo code, we’ve set the sampling period to be every 10 milliseconds, which will collect 100 samples/second. That’s enough to provide insight over a minute or so, but also happens infrequently enough so it doesn’t add noticeable overheads.</p><p>After deploying our BPF code, we have to collect the results from the BPF maps. We access the maps from user-space using the BCC APIs:</p><pre><code class="language-cpp">ebpf::BPFStackTable stack_traces =
     bcc-&gt;get_stack_table(kStackTracesMapName);

ebpf::BPFHashTable&lt;stack_trace_key_t, uint64_t&gt; histogram =
     bcc-&gt;get_hash_table&lt;stack_trace_key_t, uint64_t&gt;(kHistogramMapName);
</code></pre><p>Finally, we want to convert our addresses to symbols, and to concatenate our user and kernel stack traces. Fortunately, BCC has once again made our life easy on this one. In particular, there is a call <code>stack_traces.get_stack_symbol</code>, that will convert the list of addresses in a stack trace into a list of symbols. This function needs the PID, because it will lookup the debug symbols in the process’s object file to perform the translation.</p><pre><code class="language-cpp"> std::map&lt;std::string, int&gt; result;

 for (const auto&amp; [key, count] : histogram.get_table_offline()) {
   if (key.pid != target_pid) {
     continue;
   }

   std::string stack_trace_str;

   if (key.user_stack_id &gt;= 0) {
     std::vector&lt;std::string&gt; user_stack_symbols =
         stack_traces.get_stack_symbol(key.user_stack_id, key.pid);
     for (const auto&amp; sym : user_stack_symbols) {
       stack_trace_str += sym;
       stack_trace_str += &quot;;&quot;;
     }
   }

   if (key.kernel_stack_id &gt;= 0) {
     std::vector&lt;std::string&gt; user_stack_symbols =
         stack_traces.get_stack_symbol(key.kernel_stack_id, -1);
     for (const auto&amp; sym : user_stack_symbols) {
       stack_trace_str += sym;
       stack_trace_str += &quot;;&quot;;
     }
   }

   result[stack_trace_str] += 1;
 }
</code></pre><p>It actually turns out that the process of turning the stack traces into symbols is the part of this process that introduces the most overhead. The actual sampling of stack traces is negligible. Our next blog post will discuss the performance challenges of making this basic profiler ready for production.</p><h2>Running the Demo Profiler</h2><div class="image-xl"><svg xmlns="http://www.w3.org/2000/svg"></svg></div><p>The code and instructions for running the simple eBPF based profiler can be found <a href="https://github.com/pixie-io/pixie-demos/tree/main/ebpf-profiler">here</a>.</p><p>The code was designed to have as few dependencies as possible, but you need BCC installed. Follow the instructions in <a href="https://github.com/pixie-io/pixie-demos/tree/main/ebpf-profiler/README.md"><code>README.md</code></a> for more details on building the profiler and a toy app to profiler.</p><p>Once built, you can run the profiler as follows:</p><pre><code class="language-bash"># Profile the application for 30 seconds.
sudo ./perf_profiler &lt;target PID&gt; 30
</code></pre><p>You should see stack traces from the target program like the following:</p><pre><code class="language-bash">Successfully deployed BPF profiler.
Collecting stack trace samples for 30 seconds.
1 indexbytebody;runtime.funcname;runtime.isAsyncSafePoint; … ;runtime.goexit;
5 main.main;runtime.main;runtime.goexit;
24 main.sqrt;runtime.main;runtime.goexit;
</code></pre><p>You can then experiment by running it against other running processes in your system. Some processes may not have debug symbols installed, in which case you will see the <code>[UNKNOWN]</code> marker.</p><p>Another fun experiment is to run it against a program written in a different language, like Python or Java. You’ll see stack traces, but they’re probably not what you’re expecting. For example, with Python, what you’ll see is the Python interpreter’s stack traces rather than the Python application (Note: you’ll need the interpreter’s debug symbols to see the functions; On Ubuntu, you can get these by running something like <code>apt install python3.6-dbg</code>) . We’ll cover profiling for Java and interpreted languages in a future post.</p><p>In <a href="/cpu-profiling-3">part three</a> of this series, we’ll discuss the challenges we faced in building out this simple profiler into a production ready profiler.</p>
