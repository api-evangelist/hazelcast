---
title: "Announcing Hazelcast Platform 5.6.0: Enhanced Control and Resilience"
url: "https://hazelcast.com/blog/announcing-hazelcast-platform-5-6-0/"
date: "Tue, 21 Oct 2025 13:00:42 +0000"
author: "Hazelcast Comms"
feed_url: "https://hazelcast.com/feed/"
---
<p>We are happy to announce the general availability of Hazelcast Platform 5.6.0 for our Community and Enterprise customers. Based on real-world feedback from our community, this release strengthens our core platform capabilities and improves its management and reliability.</p><img alt="" class="attachment-post-single-inserted size-post-single-inserted wp-post-image" height="600" src="https://hazelcast.com/wp-content/uploads/2025/10/5-6-0-announcement-tile.png" width="1338" />
<p>Hazelcast Platform 5.6.0 delivers:</p>
<ul>
<li><strong>CP Subsystem Enhancements:</strong> Snapshot chunking, serialization, and log synchronization enhancements provide greater reliability, higher performance, and improved user experience.</li>
<li><strong>Vector Collection Enhancements (Beta):</strong> Optimized index handling, CPU-aware search execution, and enhanced fault tolerance.</li>
<li><strong>Dynamic Diagnostic Logging (Beta):</strong> Diagnostic logging can now be enabled without cluster restarts. </li>
<li><strong>Platform Hardening and Improvements:</strong> High-Density Memory Store optimizations for higher throughput and addition of new network queue metrics for proactive system health monitoring.</li>
</ul>
<p>These platform enhancements are delivered in concert with new versions of our ecosystem tools, <strong>Hazelcast Management Center 5.9.0</strong> and the <strong>Hazelcast Platform Operator 5.16</strong>, that support these new features.</p>
<p>Hazelcast Platform version 5.6.0 of our open source Community Edition incorporates fixes and Common Vulnerabilities and Exposures (CVE) patches since the last minor release. Enterprise customers have access to CVE patches as soon as they are ready to ensure timely compliance with their industry’s regulatory requirements.  </p>
<p>For full details, please review the Hazelcast Platform 5.6.0 <a href="https://docs.hazelcast.com/hazelcast/5.6/release-notes/community#5-6-0">Community Edition</a> and <a href="https://docs.hazelcast.com/hazelcast/5.6/release-notes/enterprise#5-6-0">Enterprise Edition</a> release notes.</p>
<h2>Enhancing CP Subsystem (Enterprise)</h2>
<p>At the heart of many mission-critical applications lies the need for strong data consistency. Hazelcast&#8217;s CP Subsystem provides this guarantee by implementing the industry-standard Raft consensus protocol. It is the foundation for data structures like CP FencedLock, CP Maps, and IAtomicLong, where correctness and consistency are paramount. </p>
<p>Key CP Enhancements include:</p>
<h3>Smart, Incremental Snapshot Transfers</h3>
<p>In the Raft protocol, a leader member sends a data snapshot to follower members to keep them in sync, especially when a follower falls behind due to network issues or membership changes. Previously, this snapshot was sent as a single, monolithic message. This could lead to memory pressure, network congestion, and lengthy synchronization times for large datasets. With the continued adoption of CP data structures, we are improving how we do snapshot transmissions, which will enhance the resilience of this core component.</p>
<p>With <strong>CP Snapshot Chunking</strong>, the leader member breaks the large snapshot into smaller, fixed-size chunks before transmission. It then sends these chunks sequentially to the follower, which receives and reassembles them to reconstruct the full snapshot. This new feature reduces network congestion, lowers tail latencies, and improves overall speed and cluster performance for large datasets.  </p>
<p><em>Note:  To ensure cluster-wide compatibility and correctness, this feature requires that <strong>all cluster members</strong> are running Hazelcast Platform 5.6.0 or later.</em></p>
<h4>CP Subsystem Performance and Usability Optimizations</h4>
<p>Alongside CP Snapshot Chunking, Platform 5.6.0 delivers several optimizations that further enhance the CP Subsystem efficiency and user experience:</p>
<ul>
<li>CP snapshot serialization optimization, which provides lower memory usage.</li>
<li>Log synchronization improvements deliver faster recovery and improved availability of CP groups.</li>
<li>CP Smart Client enables clients to route requests directly to CP group leaders, improving the performance of CP operations.</li>
</ul>
<p>To learn more about these and other CP Subsystem enhancements, visit the <a href="https://docs.hazelcast.com/hazelcast/5.6/release-notes/enterprise#5-6-0">Platform 5.6.0 Enterprise Edition release notes</a>.</p>
<h2>Enhancing Vector Collection (Beta) (Enterprise)</h2>
<p>The <a href="https://docs.hazelcast.com/hazelcast/5.6/data-structures/vector-search-overview">vector collection</a> data structure (beta) was introduced in Hazelcast Platform 5.5 Enterprise Edition. With the 5.6 release, we are delivering the following enhancements: </p>
<ul>
<li><strong>Fault Tolerance:</strong> Split-brain protection and support for synchronous and asynchronous backups.</li>
<li><strong>Enhanced Performance:</strong> Configurable dedicated thread pool for vector searches sized according to the host CPU.</li>
<li><strong>Greater Accuracy:</strong> New <a href="https://docs.hazelcast.com/hazelcast/5.6/data-structures/vector-search-overview#tuning-tips:~:text=Adjust%20efSearch%20to,decrease%20in%20precision.">efSearch</a> parameter enables users to achieve the desired balance between throughput/latency and precision. </li>
<li><strong>Deeper Observability:</strong> Metrics added to provide greater operational insights.</li>
</ul>
<p>To learn more about all the improvements, check out the <a href="https://docs.hazelcast.com/hazelcast/5.6/release-notes/enterprise#5-6-0:~:text=Vector%20Search%20(BETA)%20enhancements%3A">Platform 5.6.0 Enterprise Edition release notes</a>.</p>
<h2>Turn On Diagnostic Logging—Without Restarting Your Cluster (Beta)</h2>
<p>Observability is critical for troubleshooting issues in distributed systems. Previously, enabling Hazelcast&#8217;s powerful diagnostic logging required member restarts. While the rolling restart capability mitigates downtime concerns, there may still be resource impact from data repartitioning. Furthermore, restarting can clear the conditions that cause transient issues, making it difficult to diagnose.</p>
<p>Hazelcast Platform 5.6.0 introduces Dynamic Diagnostic Logging to solve this. This feature enables you to update its logging configuration <strong>without restarts</strong>. When you detect an anomaly or investigate a performance issue, you can immediately, with one click, enable verbose logging to capture the necessary data. Once collected, you can disable it just as easily with no service impact, enabling more efficient issue resolution.  An auto-shutoff timer allows you to enable diagnostics logging for a predetermined period, which ensures it is automatically terminated.</p>
<p>We have also integrated the management and configuration of this new feature directly into the tools you already use to manage your Hazelcast deployments, ensuring a seamless and intuitive experience.</p>
<h4>Diagnostic Logging via Management Center (MC 5.9.0+)</h4>
<p>Hazelcast Management Center 5.9.0 supports diagnostic logging for users who prefer a graphical interface. From the interface, you can:</p>
<ul>
<li>View the current state of diagnostic logging.</li>
<li>Toggle the service on or off with a single click.</li>
<li>Set the auto-disable timer to a desired duration.</li>
<li>Control the output location of the logs (e.g., file or standard output).</li>
</ul>
<p>For those who rely on automation and scripting, these controls are also exposed via the Management Center REST API, allowing you to integrate diagnostic control into your existing operational workflows and CI/CD pipelines.</p>
<p><img alt="Diagnostic Logging with Management Center" class="size-full wp-image-42958 aligncenter" height="1329" src="https://hazelcast.com/wp-content/uploads/2025/10/management-center-diagnostic-logging.png" width="1999" /></p>
<p>For complete details, see the <a href="https://docs.hazelcast.com/management-center/5.9/release-notes/releases">Management Center 5.9.0 release notes</a>.</p>
<h4>Diagnostic Logging via Hazelcast Platform Operator for Kubernetes (Operator 5.16+) (Enterprise)</h4>
<p>For the growing number of customers deploying Hazelcast on Kubernetes, we have extended the Hazelcast Platform Operator (version 5.16 and later) to support diagnostic logging configuration as code. For complete details, see the<a href="https://docs.hazelcast.com/operator/5.16/release-notes"> Hazelcast Operator 5.16 release notes</a>.</p>
<h2>Continuous Platform Hardening and Improvements</h2>
<p>Beyond the new features, other key customer-driven enhancements in this release include:</p>
<ul>
<li><strong>High-Density (HD) Memory Optimization (Enterprise):</strong>  Optimized memory allocation provides as high as a 30% throughput increase for HD IMaps with global index use cases.</li>
<li><strong>Improved Observability:</strong> Diagnostic network queue sizes are now exposed as standard metrics, which provide deeper visibility into the health of the Hazelcast networking layer, helping to proactively identify and diagnose potential backpressure issues before they impact the system.</li>
</ul>
<h2>Getting Started with Hazelcast Platform 5.6.0</h2>
<ul>
<li>Download <a href="https://hazelcast.com/community-edition-projects/downloads/?utm_source=docs-website">Hazelcast Platform Community Edition</a> and follow the installation instructions <a href="https://docs.hazelcast.com/hazelcast/5.6/getting-started/install-hazelcast">here</a>.</li>
<li>New to Hazelcast Enterprise? Download <a href="https://hazelcast.com/get-started/download/">Hazelcast Platform Enterprise Edition</a> and follow the <a href="https://docs.hazelcast.com/hazelcast/latest/getting-started/get-started-enterprise">getting started instructions</a> to unlock our Enterprise features using our <a href="https://hazelcast.com/get-started/">free trial license</a>.</li>
<li>Explore using the latest <a href="https://hazelcast.com/community-edition-projects/downloads/#hazelcast-management-center">Hazelcast Management Center 5.9.0</a>, our free GUI tool to manage your deployments and monitor your Hazelcast clusters.</li>
<li>Read the complete details in the <a href="https://docs.hazelcast.com/hazelcast/5.6/release-notes/releases">Hazelcast Platform 5.6.0 release notes</a>.</li>
<li><a href="https://hazelcast.com/contact/">Contact us</a> for further questions or to see a demo.</li>
</ul>
<p>The post <a href="https://hazelcast.com/blog/announcing-hazelcast-platform-5-6-0/">Announcing Hazelcast Platform 5.6.0: Enhanced Control and Resilience</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
