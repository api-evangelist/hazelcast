---
title: "Hazelcast Takes the Lead for Spring Session Hazelcast"
url: "https://hazelcast.com/blog/spring-session-hazelcast/"
date: "Tue, 14 Oct 2025 13:45:24 +0000"
author: "Hazelcast Comms"
feed_url: "https://hazelcast.com/feed/"
---
<p>As announced in the Spring blog <a href="https://spring.io/blog/2025/10/14/spring-session-hazelcast-new-leadership">post</a> <em>“Spring Session Hazelcast: Now Led by Hazelcast Team,”</em> we’re excited to share that the <strong>Spring Session Hazelcast</strong> module will now be maintained and developed by the <strong><a href="https://hazelcast.com/">Hazelcast</a> engineering team</strong>—the same team behind the in-memory data grid and real-time stream processing platform trusted by developers and enterprises worldwide.</p>
<p>As part of that effort, we’ve collaborated with the Spring engineering team to ensure a <strong>seamless transition</strong> and continued support for the thousands of developers who rely on Hazelcast for session management within their Spring applications.</p>
<h2>Continuity for the Spring Developer Community</h2>
<p>For over a decade, <strong>Spring Session</strong> has made it easy to replace and customize session management in a vendor-agnostic way. Developers can implement <a href="https://docs.spring.io/spring-session/reference/api.html#api-sessionrepository">SessionRepository</a> to plug in any backing store — and Hazelcast has long been a popular choice for its simplicity, scalability, and performance.</p>
<p>By taking ownership of the module, <strong>Hazelcast ensures that this integration remains actively supported, tested, and improved</strong>—without disruption to current users. Going forward, new generations of <strong>Spring Session Hazelcast</strong> will be released from a <strong>new GitHub repository</strong> and published under <strong>new Maven coordinates</strong>, maintained by the Hazelcast team.</p>
<h2>What This Means for Developers</h2>
<p>If you’re currently using Spring Session with Hazelcast:</p>
<ul>
<li>Your existing setup will continue to work as-is.</li>
<li>Future updates and fixes will be available directly from Hazelcast.</li>
<li>You can expect continued <strong>support</strong> for the <strong>latest Spring releases</strong>, along with <strong>ongoing improvements</strong> to the user experience, documentation, and testing.</li>
<li>New releases will use new groupId and package name.</li>
</ul>
<p>This transition reflects Hazelcast’s ongoing investment in the Java ecosystem and our commitment to open collaboration with Spring’s developer community.</p>
<h2>Why This Matters</h2>
<p>Hazelcast has been a trusted integration partner in the Spring ecosystem for many years—across <strong>Spring Session</strong>, <strong>Spring Integration</strong>, and <strong>Spring Data</strong>. Taking stewardship of Spring Session Hazelcast allows us to:</p>
<ul>
<li><strong>Strengthen our relationship</strong> with millions of Java developers who use Hazelcast for distributed caching and state management.</li>
<li><strong>Reinforce our commitment</strong> to open-source innovation, ensuring Hazelcast remains a first-class citizen within the Spring ecosystem.</li>
</ul>
<h2>Upcoming Changes</h2>
<ul>
<li><strong>Keep using the current artifact (for now):</strong><br />
Use the existing Maven coordinates until the Hazelcast-led artifact is available:</p>
<pre class="EnlighterJSRAW">&lt;groupId&gt;org.springframework.session&lt;/groupId&gt;
&lt;artifactId&gt;spring-session-hazelcast&lt;/artifactId&gt;
&lt;version&gt;3.5.2&lt;/version&gt;</pre>
</li>
<li><strong>Source code location:</strong> The project is moving to a new repository: <a href="https://github.com/hazelcast/hazelcast-spring-session">https://github.com/hazelcast/hazelcast-spring-session</a></li>
<li><strong>Future Maven coordinates:</strong> Hazelcast will publish under the com.hazelcast groupId. We’ll update the README in the new repo with full details by the <strong>end of October 2025</strong>.</li>
<li><strong>What’s next:</strong> Stay tuned for roadmap updates and ways to get involved with the community.</li>
</ul>
<p>We’d like to thank the <strong>Spring engineering team</strong> for their collaboration and the <strong>Spring community</strong> for their continued trust in Hazelcast. Together, we’re ensuring that Spring developers can continue to build <strong>high-performance</strong>, <strong>distributed applications</strong> backed by the reliability of Hazelcast.</p>
<p>For more details, see the Spring <a href="https://spring.io/blog/2025/10/14/spring-session-hazelcast-new-leadership">announcement</a> or visit our GitHub repository (<a href="https://github.com/hazelcast/hazelcast-spring-session">link</a>) for migration information and future updates.</p>
<p>The post <a href="https://hazelcast.com/blog/spring-session-hazelcast/">Hazelcast Takes the Lead for Spring Session Hazelcast</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
