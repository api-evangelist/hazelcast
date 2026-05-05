---
title: "Turbocharging Java Reflection Performance with MethodHandle"
url: "https://hazelcast.com/blog/turbocharging-java-reflection-performance-with-methodhandle/"
date: "Wed, 09 Apr 2025 15:00:48 +0000"
author: "Jack Green"
feed_url: "https://hazelcast.com/feed/"
---
<p><span style="font-weight: 400;">Reflection in Java is a powerful feature that allows introspection and manipulation of the application at runtime. However, it comes with a well-known drawback: </span><a href="https://docs.oracle.com/javase/tutorial/reflect/index.html#:~:text=Performance%20Overhead"><span style="font-weight: 400;">performance overhead</span></a><span style="font-weight: 400;">:</span></p>
<p><i><span style="font-weight: 400;">&#8220;Reflection involves types that are dynamically resolved, certain Java virtual machine optimizations can not be performed. Consequently, reflective operations have slower performance than their non-reflective counterparts&#8221;</span></i></p>
<p><span style="font-weight: 400;">Developers often accept this cost as unavoidable, but what if there was a way to mitigate it?</span></p>
<p><span style="font-weight: 400;">In this blog post, we’ll explore how we investigated and optimized reflection performance, specifically in the context of Hazelcast metrics collection. We&#8217;ll cover:</span></p>
<ul>
<li style="font-weight: 400;"><span style="font-weight: 400;">The problem and context – how Hazelcast uses reflection in metrics gathering</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Investigation and benchmarking – measuring the impact of using reflection</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Key takeaways and next steps– how these findings were applied in Hazelcast</span></li>
</ul>
<p><span style="font-weight: 400;">Read on if you&#8217;ve ever wondered whether you can improve the efficiency of your Java applications&#8217; reflection.</span></p>
<h3><strong>Problem and Context</strong></h3>
<p><span style="font-weight: 400;">To support monitoring, </span><a href="https://docs.hazelcast.com/hazelcast/latest/maintain-cluster/monitoring#metrics"><span style="font-weight: 400;">Hazelcast Platform can collect metrics</span></a><span style="font-weight: 400;"> to be supplied to various consumers, which</span><span style="font-weight: 400;"> is implemented internally by </span><a href="https://github.com/hazelcast/hazelcast/blob/v5.5.0/hazelcast/src/main/java/com/hazelcast/internal/partition/impl/MigrationStats.java#L158-L165"><span style="font-weight: 400;">annotating the field or method that reports the metric</span></a><span style="font-weight: 400;"> and then using reflection to collect all those values.</span></p>
<p><span style="font-weight: 400;">For methods, </span><a href="https://github.com/hazelcast/hazelcast/blob/v5.3.0/hazelcast/src/main/java/com/hazelcast/internal/metrics/impl/MethodProbe.java#L135-L152"><span style="font-weight: 400;">this lookup </span><i><span style="font-weight: 400;">was</span></i><span style="font-weight: 400;"> implemented</span></a><span style="font-weight: 400;"> by finding the annotated methods and simply </span><span style="color: green; font-family: Consolas, Monaco, monospace;">invoke</span><span style="font-weight: 400;">-ing them:</span></p>
<pre class="EnlighterJSRAW">Long result = (Long) method.invoke(source);
return result.longValue();</pre>
<p><span style="font-weight: 400;">This approach works fine &#8211; but can we do better?</span></p>
<h3><strong>Investigation and Benchmarking</strong></h3>
<p><span style="font-weight: 400;">Let’s put some numbers together with a </span><a href="https://github.com/openjdk/jmh"><span style="font-weight: 400;">JMH</span></a><span style="font-weight: 400;"> benchmark, comparing our simple reflective method call versus a direct method call to quantify the overhead.</span></p>
<pre><span style="color: green;">mvn clean install jmh:benchmark </span>
<span style="color: green;">-Djmh.benchmarks=benchmark.PrimitiveReflectiveAccessBenchmark</span></pre>
<table style="width: 561px; height: 164px;">
<thead>
<tr>
<th>
<p style="text-align: center;"><strong>Access Type</strong></p>
</th>
<th>
<p style="text-align: center;"><strong>Execution Time <br />
(nanoseconds/operation)</strong></p>
</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0.5</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.5</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">So, we’ve quantified in this (synthetic) example that reflection is over 10x slower.</span></p>
<p><a href="https://dev.java/learn/introduction_to_method_handles/"><span style="font-weight: 400;">MethodHandles</span></a><span style="font-weight: 400;"> are (as the name suggests) a directly invocable reference to a method and offer improved performance by </span><a href="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/MethodHandles.Lookup.html#access"><span style="font-weight: 400;">moving some accessibility checks</span></a><span style="font-weight: 400;"> from being evaluated when the method is </span><span style="color: green; font-family: Consolas, Monaco, monospace;">invoke</span>d<span style="font-weight: 400;"> to when the </span><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span><span style="font-weight: 400;"> is </span><i><span style="font-weight: 400;">created.</span></i><span style="font-weight: 400;"> By keeping a reference to the MethodHandle we can do these checks just once rather than every time it’s </span><span style="font-weight: 400;">invoke</span><span style="font-weight: 400;">d,  as with our original reflective implementation.</span></p>
<p><span style="font-weight: 400;">First, let’s swap our use of simple Java reflection with </span><a href="https://dev.java/learn/introduction_to_method_handles/"><span style="font-weight: 400;">MethodHandles</span></a><span style="font-weight: 400;">.</span></p>
<p><span style="font-weight: 400;">Obtaining the <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span><span style="font-weight: 400;">:</span></p>
<pre class="EnlighterJSRAW">Method method = source.getClass().getDeclaredMethod("longMethod");
MethodHandle methodHandle = MethodHandles.lookup().unreflect(method);</pre>
<p><span style="font-weight: 400;">And then <span style="color: green; font-family: Consolas, Monaco, monospace;">invoke</span></span><span style="font-weight: 400;">-ing the same method as before:</span></p>
<pre class="EnlighterJSRAW">Long result = (Long) methodHandle.invoke(source);
return result.longValue();</pre>
<p><span style="font-weight: 400;">Re-running our benchmarks:</span></p>
<table style="width: 488px; height: 169px;">
<tbody>
<tr>
<td>
<p style="text-align: center;"><strong>Access Type</strong></p>
</td>
<td>
<p style="text-align: center;"><span style="font-weight: 400;"><strong>Execution Time</strong><br />
<strong>(nanoseconds/operation</strong>)</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0.5</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.5</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle invoke</span></span></p>
</td>
<td>
<p><span style="font-weight: 400;">     4.1</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">Now we see a significant improvement &#8211; over 50% faster.</span></p>
<p><span style="font-weight: 400;">However, performance isn’t just runtime &#8211; we also need to consider memory allocation. Although the method <span style="color: green; font-family: Consolas, Monaco, monospace;">return</span></span><span style="font-weight: 400;"> is a </span><a href="https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html"><span style="font-weight: 400;">primitive</span></a> <span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">long</span></span><span style="font-weight: 400;">, both our reflective operations are getting a </span><a href="https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html"><span style="font-weight: 400;">boxed</span></a> <span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">Long</span></span><span style="font-weight: 400;"> object and then having to unbox it to get the </span><span style="font-weight: 400;">long</span><span style="font-weight: 400;"> primitive we actually want.</span></p>
<p><span style="font-weight: 400;">Luckily, JMH can measure this, too:</span></p>
<pre><span style="color: green;">mvn clean install jmh:benchmark </span>
<span style="color: green;">-Djmh.benchmarks=benchmark.PrimitiveReflectiveAccessBenchmark </span>
<span style="color: green;">-Djmh.prof=gc</span></pre>
<table style="width: 629px; height: 226px;">
<tbody>
<tr>
<td style="text-align: center;">
<p><strong>Access Type</strong></p>
</td>
<td style="text-align: center;">
<p><strong>Execution Time<br />
(nanoseconds/operation)</strong></p>
</td>
<td>
<p style="text-align: center;"><strong>Object Allocations <br />
(bytes/operation)</strong></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0.5</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.5</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     24</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle invoke</span></span></p>
</td>
<td>
<p><span style="font-weight: 400;">     4.1</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     24</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">Now we can see that these extra Object allocations are significant &#8211; each reference must be managed and garbage collected.</span></p>
<p><span style="font-weight: 400;">To address this, we can use a special behavior of <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span><span style="font-weight: 400;">:  </span><a href="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/MethodHandle.html#sigpoly"><i><span style="font-weight: 400;">signature polymorphism</span></i></a><span style="font-weight: 400;">.</span></p>
<p><span style="font-weight: 400;">This allows us to invoke the <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span>, specifying the return type (by inference) <i>even for primitive value</i>s:<br />
</span></p>
<pre class="EnlighterJSRAW">return (long) methodHandle.invokeExact(source);</pre>
<p><span style="font-weight: 400;">This </span><i><span style="font-weight: 400;">looks</span></i><span style="font-weight: 400;"> pretty similar, but when we benchmark…</span></p>
<table style="width: 760px; height: 338px;">
<tbody>
<tr>
<td>
<p style="text-align: center;"><strong>Access Type</strong></p>
</td>
<td style="text-align: center;">
<p><strong>Execution Time<br />
(nanoseconds/operation)</strong></p>
</td>
<td>
<p style="text-align: center;"><strong>Object Allocations<br />
(bytes/operation)</strong></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0.5</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.5</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     24</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle Invoke Long</span> </span><span style="font-weight: 400;">object</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     4.1</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     24</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle invokeExact Long</span> </span><span style="font-weight: 400;">primitive</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     2.7</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     0</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">Now, we can see a </span><b>significant</b><span style="font-weight: 400;"> difference &#8211; over twice as fast as reflection and orders of magnitude more efficient regarding object allocations.</span></p>
<figure class="wp-caption aligncenter" id="attachment_40437" style="width: 750px;"><img alt="" class="wp-image-40437" height="450" src="https://hazelcast.com/wp-content/uploads/2025/04/1.png" width="750" /><figcaption class="wp-caption-text" id="caption-attachment-40437"><em>Who doesn&#8217;t love a good chart?</em></figcaption></figure>
<p><span style="font-weight: 400;">Again, a <a href="https://github.com/hazelcast/java-reflection-performance-blog-benchmarks/blob/main/src/test/java/benchmark/ReferenceReflectiveAccessBenchmark.java">JMH benchmark is created</a> to compare the different access options.</span></p>
<pre><span style="color: green;">mvn clean install jmh:benchmark</span>
<span style="color: green;">-Djmh.benchmarks=benchmark.ReferenceReflectiveAccessBenchmark
</span></pre>
<table style="width: 400px; height: 214px;">
<tbody>
<tr>
<td style="text-align: center;">
<p><strong>Access Type</strong></p>
</td>
<td>
<p style="text-align: center;"><strong>Execution Time<br />
(nanoseconds/operation)</strong></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     2.1</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.0</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span></p>
</td>
<td>
<p><span style="font-weight: 400;">     3.6</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">As before, <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span><span style="font-weight: 400;"> is offering increased performance &#8211; but we can use these in </span><i><span style="font-weight: 400;">conjunction</span></i><span style="font-weight: 400;"> with </span><a href="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/LambdaMetafactory.html"><span style="font-weight: 400;">LambdaMetafactory</span></a><span style="font-weight: 400;"> to do </span><b>even</b><span style="font-weight: 400;"> better.</span></p>
<p><span style="font-weight: 400;">Using <span style="color: green; font-family: Consolas, Monaco, monospace;">LambdaMetafactory</span></span><span style="font-weight: 400;">, we can optimize further by</span> <i><span style="font-weight: 400;">directly binding</span></i> <span style="font-weight: 400;">the target of the</span><span style="font-weight: 400;"> <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span> </span><span style="font-weight: 400;">into a lambda</span><span style="font-weight: 400;"> <span style="color: green; font-family: Consolas, Monaco, monospace;">Function</span></span><span style="font-weight: 400;">, </span><b>avoiding any reflective access altogether at runtime</b><span style="font-weight: 400;">.</span></p>
<pre class="EnlighterJSRAW">MethodHandles.Lookup lookup = MethodHandles.lookup();
 
// The accessor method we want to access
Method method = source.getClass().getDeclaredMethod("getLong");
 
MethodHandle methodHandle = lookup.unreflect(method);
 
// The type we want to bind our accessor to
MethodType factoryType = MethodType.methodType(Function.class);
// The expected return type of the accessor and the class the accessor is contained within
MethodType interfaceMethodType = MethodType.methodType(Long.class, source.getClass());
 
// A reference to the accessor via the Function intermediary
CallSite callSite = LambdaMetafactory.metafactory(lookup, "apply", factoryType, interfaceMethodType, methodHandle, methodHandle.type());
 
// The resultant Function that's now bound to the accessor
Function&lt;SomeSource, Long&gt; lambdaMetafactoryFunction = (Function&lt;SomeSource, Long&gt;) callSite.getTarget().invoke();</pre>
<p><span style="font-weight: 400;">This dynamically generates a lambda equivalent to:<br />
</span></p>
<pre class="EnlighterJSRAW">Function&lt;SomeSource, Long&gt; myFunction = new Function&lt;SomeSource, Long&gt;() {
    @Override
    public Long apply(SomeSource someSource) {
        return someSource.getLong();
    }
};
</pre>
<p>&nbsp;</p>
<table style="width: 400px; height: 289px;">
<tbody>
<tr>
<td style="text-align: center;">
<p><strong>Access Type</strong></p>
</td>
<td>
<p style="text-align: center;"><strong>Execution Time<br />
(nanoseconds/operation)</strong></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Direct</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     2.2</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;">Simple Reflective Call</span></p>
</td>
<td>
<p><span style="font-weight: 400;">     6.0</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span></p>
</td>
<td>
<p><span style="font-weight: 400;">     3.6</span></p>
</td>
</tr>
<tr>
<td>
<p><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">LambdaMetafactory</span></span></p>
</td>
<td>
<p><span style="font-weight: 400;">     2.2</span></p>
</td>
</tr>
</tbody>
</table>
<p><span style="font-weight: 400;">Again, we are seeing a </span><b>significant</b><span style="font-weight: 400;"> performance improvement &#8211; matching direct access speeds but accessing dynamically at runtime.</span></p>
<p><img alt="" class="aligncenter size-full wp-image-40485" height="450" src="https://hazelcast.com/wp-content/uploads/2025/04/SecondBlogGraphV2.svg" width="750" /></p>
<h2><span style="font-weight: 400;">Key Takeaways and Next Steps</span></h2>
<p><span style="font-weight: 400;">Our investigation demonstrates that the perceived performance cost of Java reflection isn&#8217;t insurmountable. By strategically replacing standard reflection calls with <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span><span style="font-weight: 400;">s and <span style="color: green; font-family: Consolas, Monaco, monospace;">LambdaMetafactory</span></span><span style="font-weight: 400;">, we achieved significant performance improvements, even matching direct access speeds in some cases.</span></p>
<p><span style="font-weight: 400;">Key takeaways from this investigation include:</span></p>
<ul>
<li style="font-weight: 400;"><b><span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span></span> and <span style="font-weight: 400;"><span style="color: green; font-family: Consolas, Monaco, monospace;">LambdaMetafactory</span></span></b><b> Offer Real Gains:</b><span style="font-weight: 400;"> These APIs provide powerful mechanisms to optimize reflective operations, reducing overhead associated with dynamic lookups and enabling JVM optimizations. <span style="color: green; font-family: Consolas, Monaco, monospace;">invokeExact</span></span><span style="font-weight: 400;"> proved particularly effective for primitive types by avoiding boxing/unboxing costs.  </span></li>
<li style="font-weight: 400;"><b>Benchmarking is Crucial:</b><span style="font-weight: 400;"> Quantifying performance differences using tools like JMH was essential to identify bottlenecks and validate the effectiveness of our optimizations.  </span></li>
<li style="font-weight: 400;"><b>Real-World Impact:</b><span style="font-weight: 400;"> These optimizations weren&#8217;t just theoretical; they were successfully integrated into Hazelcast 5.4, enhancing the efficiency of its metrics collection system.</span></li>
</ul>
<p><span style="font-weight: 400;">For Java developers grappling with reflection-related performance bottlenecks, adopting <span style="color: green; font-family: Consolas, Monaco, monospace;">MethodHandle</span> and <span style="color: green; font-family: Consolas, Monaco, monospace;">LambdaMetafactory</span></span><span style="font-weight: 400;"> presents a viable path toward more efficient and responsive applications. Careful benchmarking is key to realizing these gains in your specific context.</span></p>
<p><span style="font-weight: 400;">Want to explore further? Check out </span><a href="https://github.com/hazelcast/java-reflection-performance-blog-benchmarks"><span style="font-weight: 400;">the benchmarks</span></a><span style="font-weight: 400;"> and </span><a href="https://github.com/hazelcast/hazelcast/pull/25279"><span style="font-weight: 400;">Hazelcast Implementation</span></a><span style="font-weight: 400;">.</span></p>
<p>The post <a href="https://hazelcast.com/blog/turbocharging-java-reflection-performance-with-methodhandle/">Turbocharging Java Reflection Performance with MethodHandle</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
