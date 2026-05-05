---
title: "Hazelcast Spring Session is getting a refresh (and Hazelcast now maintains it)"
url: "https://hazelcast.com/blog/hazelcast-spring-session-is-getting-a-refresh/"
date: "Thu, 19 Feb 2026 16:00:45 +0000"
author: "Tomasz Gawęda"
feed_url: "https://hazelcast.com/feed/"
---
<p>If you use Spring Session with Hazelcast to share HTTP sessions across multiple application instances, you already know the value it brings: scalability, resilience, and performance. You may also be familiar with some of the sharp edges—configuration that must be just right, subtle issues around serialization, and problems that only surface at scale or when moving to client/server architectures.</p>
<p>We’re changing that.</p>
<p>In October 2025, the <strong>Hazelcast team took over stewardship of Spring Session Hazelcast </strong>(<a href="https://hazelcast.com/blog/spring-session-hazelcast/">read our blog announcement here</a>). This ensures the integration remains actively supported, aligned with modern Hazelcast and Spring practices, and continues to evolve with the needs of users. </p>
<p>This post explains what we’re modernizing, and what you can expect in the upcoming <strong>Hazelcast Spring Session 4.0</strong> release.</p>
<h2 class="h3">What’s changing in 4.0 (and why it’s a major version)</h2>
<p><strong>Hazelcast Spring Session 4.0</strong> is the first release led by the Hazelcast team. To reflect the new ownership and module home, it includes <strong>backwards-incompatible changes</strong> to package names and Maven coordinates. </p>
<p>The migration notes in the repo cover the key updates, including the new groupId/artifactId and moved packages: [<a href="https://docs.hazelcast.com/hazelcast/latest/spring/hazelcast-spring-session#migrate-from-spring-session-hazelcast-3-x">Configure Hazelcast Spring Session | Hazelcast Docs</a>]</p>
<p>This major bump also gave us room to improve the developer experience in areas that previously couldn’t be changed safely—especially serialization: We made a breaking change that allows you to use client-server architecture without any need for server-side code deployment and we’ve updated the serialization configuration so that it is more unified between deployment styles.</p>
<h3 class="h4">Developer Experience improvements</h3>
<p>Hazlecast Spring Session has long been a popular backing store for Spring sessions because of its simplicity, scalability, and performance. With 4.0, we’re building on those strengths while making the module easier to configure, safer to run in production and clearer to reason about.</p>
<p><strong>1) Session map auto-configuration</strong></p>
<p>Today, many users must manually configure the same map attributes and indexes to ensure efficient lookups (for example, indexing the principal name). In 4.0, the module will <strong>auto-configure sensible defaults</strong> when you haven’t provided your own, reducing repetitive and error-prone setup. </p>
<p><strong>2) Safer default serialization setup</strong></p>
<p>Currently, inconsistent serializer configuration across nodes can lead to failures that are hard to diagnose. We’re moving toward <strong>out-of-the-box serializer registration</strong> so the “works on one node, fails on another” class of problems goes away. This also aligns better with Hazelcast’s recommended serialization patterns. </p>
<p><strong>3) Improved client/server experience</strong></p>
<p>In client/server deployments, requiring application classes and serializers can be difficult to manage. 4.0 <strong>improves how session attributes can be handled</strong> so fewer server-side classes and custom serializers are required, making client/server architectures easier to operate and evolve.</p>
<p><strong>4) A complete, explicit configuration guide</strong></p>
<p>We’re investing in the documentation to make <strong>onboarding and configuration straightforward</strong> and clear. Updates include:</p>
<ul>
<li>Required vs. optional index configuration</li>
<li>Serialization behaviour: what runs on clients vs. members, and why</li>
<li>A complete reference of configuration parameters</li>
<li>What is auto-configured, when it kicks in, and how to override it</li>
</ul>
<h2 class="h3">Who this is for</h2>
<p>This modernization is aimed at developers who:</p>
<ul>
<li>Run one or more Spring Boot instances and need resilient, fault-tolerant session storage</li>
<li>Are moving from local sessions to clustered sessions and want the simplest path</li>
<li>Run Hazelcast in <strong>client/server</strong> mode and want fewer classpath/serializer surprises</li>
<li>Care about <strong>clear, explicit docs</strong> and production-ready defaults</li>
</ul>
<h2 class="h3">Where to follow along (and how to prepare)</h2>
<ul>
<li><a href="https://github.com/hazelcast/hazelcast-spring-session">Repo</a> + <a href="https://docs.hazelcast.com/hazelcast/latest/spring/hazelcast-spring-session#migrate-from-spring-session-hazelcast-3-x">migration notes</a>: the Hazelcast-led module lives here, including what changes in 4.0. </li>
<li><a href="https://hazelcast.com/blog/spring-session-hazelcast/">Hazelcast announcement</a>: background on Hazelcast taking the lead. </li>
<li><a href="https://spring.io/blog/2025/10/14/spring-session-hazelcast-new-leadership">Spring announcement</a>: Spring Session Hazelcast leadership transition.  </li>
<li><a href="https://hazelcast.com/blog/resilient-user-sessions-are-easier-than-you-think-with-hazelcast-spring-session/">Resilient user sessions are easier than you think</a> &#8211; blog post with hands-on example on Hazelcast Spring Session</li>
</ul>
<p>The post <a href="https://hazelcast.com/blog/hazelcast-spring-session-is-getting-a-refresh/">Hazelcast Spring Session is getting a refresh (and Hazelcast now maintains it)</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
