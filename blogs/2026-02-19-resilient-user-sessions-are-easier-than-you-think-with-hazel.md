---
title: "Resilient user sessions are easier than you think with Hazelcast Spring Session"
url: "https://hazelcast.com/blog/resilient-user-sessions-are-easier-than-you-think-with-hazelcast-spring-session/"
date: "Thu, 19 Feb 2026 16:01:38 +0000"
author: "Tomasz Gawęda"
feed_url: "https://hazelcast.com/feed/"
---
<p>Most web applications rely on user sessions in some way &#8211; to track login state, shopping baskets, or other data that makes the experience more personal.</p>
<p>The problem is that sessions are often stored directly inside the application server. When that instance goes down &#8211; and it will &#8211; the sessions disappear with it. Load balancers help distribute traffic, but they don’t magically make session state resilient, and <a href="https://www.bbc.com/news/articles/cvgvnp77dy9o">as we’ve all seen</a>, in a regional outage it certainly won’t.</p>
<p>This year, <a href="https://hazelcast.com/blog/spring-session-hazelcast/">Hazelcast took over the stewardship of the Spring Session Hazelcast module</a>. Since then, we’ve worked to ensure compatibility with Spring Session 4.x and Spring Framework 7 / Spring Boot 4 and we’ve improved the overall developer experience. In this post, I want to show how easy it is to make your sessions resilient using Hazelcast Spring Session.</p>
<h2 class="h3">Scenario</h2>
<p>Let’s imagine a very common scenario &#8211; an online shop. </p>
<p>This shop tracks several things inside user sessions:</p>
<ul>
<li>Login status </li>
<li>Current shopping basket</li>
<li>Items in the basket</li>
<li>Quantity and price at the time items were added</li>
<li>Last 50 visited products (for recommendations)</li>
</ul>
<p>The application uses Spring Boot as its main framework &#8211; a very popular choice for Java web applications. </p>
<p>Some of the domain classes might look like this:</p>
<ul>
<li><span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">principalName</span> &#8211; Spring Security’s attribute to track the logged in user</li>
<li><span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Basket</span> represented as <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">record Basket (String principalName, List&lt;BasketItem&gt; items)</span></li>
<li><span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">record BasketItem (ProductDto product, BigDecimal orderPrice, int quantity)</span></li>
<li><span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">record ProductDto(long id, String name, BigDecimal listPrice)</span></li>
</ul>
<p>The main controller, <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">ShopController</span>, exposes endpoints to add items to the basket and to view its current contents:</p>
<pre class="EnlighterJSRAW">@RequestMapping("/basket/add")
public String addToBasket(HttpSession session, @RequestParam("items") Set&lt;Long&gt; items) {
   Basket basket = getBasketAttr(session);
   Map&lt;Long, ProductDto&gt; productDtoMap = products.getAll(items);
   productDtoMap.forEach((_, productDto) -&gt; basket.items().add(new BasketItem(productDto, productDto.listPrice(), 1)));
   session.setAttribute("basket", basket);
   return items.size() + " items added";
}
</pre>
<p>The only thing we need in the controller is a <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">HttpSession</span> parameter. If you debug the application, you’ll see that Spring wraps a <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">HazelcastSession</span> underneath &#8211; provided by the Hazelcast Spring Session module. </p>
<p>If your application uses Spring Security &#8211; and most do &#8211; integrating it with Spring Session is super easy. You only need to add session management configuration like this:</p>
<pre class="EnlighterJSRAW">.sessionManagement((sessionManagement) -&gt; sessionManagement
                          .maximumSessions(2)
                          .sessionRegistry(sessionRegistry())
                 )</pre>
<h2 class="h3">Why local sessions don’t scale</h2>
<p>Typically, sessions are stored inside the Servlet Container. You can run multiple containers, but session data isn’t replicated between them. If an instance goes down, users will lose their session, including their shopping basket.</p>
<p>For applications expecting high traffic, this isn’t acceptable &#8211; nobody likes it when a shop suddenly forgets everything in their basket because of a 502 error. </p>
<p>This is exactly the problem Hazelcast Spring Session solves.</p>
<h2 class="h3">Hazelcast Spring Session to the rescue</h2>
<p>Hazelcast Spring Session allows you to store the session data in a Hazelcast cluster, distributed across multiple members. </p>
<p>First, add the dependency to your project:</p>
<pre class="EnlighterJSRAW">&lt;dependency&gt;
    &lt;groupId&gt;com.hazelcast&lt;/groupId&gt;
    &lt;artifactId&gt;hazelcast-spring-session&lt;/artifactId&gt;
    &lt;version&gt;4.0.0&lt;/version&gt;
&lt;/dependency&gt;
</pre>
<p>Now, open your configuration class and add the following annotation: </p>
<pre class="EnlighterJSRAW">@EnableHazelcastHttpSession</pre>
<p>That’s the basic setup &#8211; yay! </p>
<p>You’ll also need to add a <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">HazelcastInstance</span> bean. Once that’s done, you can start your application and your sessions are distributed across the cluster.</p>
<p>Our session map will be called <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">shopSessions</span>, referenced by the <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">SESSION_MAP_NAME</span> constant:</p>
<pre class="EnlighterJSRAW">@EnableHazelcastHttpSession(sessionMapName = SESSION_MAP_NAME)</pre>
<h2 class="h3">Adding resilience with backups</h2>
<p>Now, let’s make the sessions resilient. </p>
<p>Configuring backups for session data in Hazelcast is as simple as:</p>
<pre class="EnlighterJSRAW">config.addMapConfig(new MapConfig(SESSION_MAP_NAME).setBackupCount(1));
</pre>
<p>This ensures each session entry is stored on another cluster member as a backup. If one node goes down, session data is still available. Pretty easy, right? </p>
<figure class="wp-caption aligncenter" id="attachment_44067" style="width: 500px;"><img alt="figure 1" class="wp-image-44067 size-full" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-1.png" width="500" /><figcaption class="wp-caption-text" id="caption-attachment-44067">Fig 1: With backups enabled, session data is replicated to another member allowing sessions to survive node failures.</figcaption></figure>
<h2 class="h3">Multi-region resilience with WAN Replication</h2>
<p>If you want to go further and protect user sessions against a full regional outage, Hazelcast Enterprise supports WAN replication (often called geo-replication) between clusters. </p>
<p>In a typical <b>active–passive </b>setup, one cluster acts as the primary location for session writes, while another cluster keeps a replicated copy. If the primary region goes down, applications can switch to the secondary cluster and continue working with existing sessions. </p>
<p>Adding WAN replication isn’t too complicated, although there’s a bit more configuration to it (after all, it’s inter-cluster communication!). A simple active-passive configuration looks like this:</p>
<pre class="EnlighterJSRAW">WanReplicationConfig wanReplicationConfig = new WanReplicationConfig();
wanReplicationConfig.setName("sessionReplication"); &lt;1&gt;
WanBatchPublisherConfig publisherConfig = new WanBatchPublisherConfig();
publisherConfig.setClusterName("us-server");
publisherConfig.setTargetEndpoints("us-server:5701"); &lt;2&gt;

wanReplicationConfig.addBatchReplicationPublisherConfig(publisherConfig);
config.addWanReplicationConfig(wanReplicationConfig);

WanReplicationRef wanReplicationRef = new WanReplicationRef();
wanReplicationRef.setName("sessionReplication");</pre>
<p>Here we configure the name of the replication (&lt;1&gt;) and a publisher (&lt;2&gt;) that contains the connection details for the target cluster.</p>
<p>Next, we attach this WAN replication configuration to the session map:</p>
<pre class="EnlighterJSRAW">config.addMapConfig(new MapConfig(SESSION_MAP_NAME)
    .setBackupCount(1)
    .setWanReplicationRef(wanReplicationRef));</pre>
<p>With this configuration, your session data is asynchronously replicated to the remote Hazelcast cluster named <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">us-server</span>.</p>
<figure class="wp-caption aligncenter" id="attachment_44069" style="width: 500px;"><img alt="Figure 2" class="wp-image-44069 size-full" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-2.png" width="500" /><figcaption class="wp-caption-text" id="caption-attachment-44069">Fig 2: With active-passive WAN replication, session updates are asynchronously replicated to a secondary region for disaster recovery.</figcaption></figure>
<h2 class="h3">Active-active replication</h2>
<p>Going one step further again, Hazelcast Enterprise also supports <b>active-active WAN replication</b>, where multiple clusters can accept writes at the same time and replicate session data to each other. This is useful when you want applications running in different regions to operate independently, while still keeping session data synchronized across clusters.</p>
<p>Let’s look at a simple <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">docker-compose.yaml</span> that starts two Hazelcast Enterprise clusters, one in London and one in New York City:</p>
<pre class="EnlighterJSRAW">services:
  london-cluster:
    image: hazelcast/hazelcast-enterprise:latest
    hostname: london-cluster-1
    ports:
      - "5701:5701"
    volumes:
      - ./conf-lon:/opt/hazelcast/config
    env_file:
      - licensekey.env
    environment:
      - HAZELCAST_CONFIG=/opt/hazelcast/config/config.yml

  nyc-cluster:
    image: hazelcast/hazelcast-enterprise:latest
    hostname: nyc-cluster-1
    ports:
      - "5702:5701"
    volumes:
      - ./conf-newyork:/opt/hazelcast/config
    env_file:
      - licensekey.env
    environment:
      - HAZELCAST_CONFIG=/opt/hazelcast/config/config.yml
</pre>
<p>The Hazelcast configuration files used by both clusters contain WAN replication settings similar to the earlier configuration. What’s important in them is that the NYC cluster points to the London cluster and vice-versa, enabling bi-directional replication. </p>
<p>Here’s a snippet from London cluster configuration showing this:</p>
<pre class="EnlighterJSRAW">wan-replication:
   # &lt;1&gt;
   replicate-to-nyc:
     batch-publisher:
       nycPublisher:
         cluster-name: nyc-cluster
         target-endpoints: "nyc-cluster-1:5701"
 map:
   shopSessions:
     wan-replication-ref:
       replicate-to-nyc:
      # &lt;2&gt; using active-active requires merge policy
         merge-policy-class-name: PutIfAbsentMergePolicy
     backup-count: 1
     # because we are modifying IMap's config not via sessionMapConfigCustomizer, we need to add the index manually
     indexes:
       - type: HASH
         attributes:
           - "principalName"
</pre>
<p>We configure standard WAN replication &lt;1&gt; and then apply it to the shopSessions map. Because both clusters can update the same session data, we must define a merge policy to resolve potential conflicts, as seen in &lt;2&gt;. In this example, <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">PutIfAbsentMergePolicy</span> ensures existing session entries are not overwritten when replicated from the other cluster, which works well for session data.</p>
<p>That’s all we need on the cluster side. The NYC configuration mirrors this setup, pointing back to the London cluster.</p>
<p>In the shop application itself, you only need to replace the <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">HazelcastInstance</span> bean declaration with a client failover configuration:</p>
<pre class="EnlighterJSRAW">@Bean
@SpringSessionHazelcastInstance
@ConditionalOnExpression("${use.wan.replication:false} == true")
public HazelcastInstance hazelcastClient() {\
// &lt;1&gt;
    ClientConfig config = new ClientConfig();
    // some configuration
    applySerializationConfig(config);
    config.setClusterName("nyc-cluster");
    // &lt;2&gt;
    ClientConfig config2 = new ClientConfig();
    // some configuration
    config2.setClusterName("london-cluster");

    // &lt;3&gt;
    return HazelcastClient.newHazelcastFailoverClient(new ClientFailoverConfig()
.addClientConfig(config2)
.addClientConfig(config)
.setTryCount(10));
    }
</pre>
<p>Here we configure connections to both clusters &lt;1&gt; and &lt;2&gt; and pass them to the failover client &lt;3&gt;. If one cluster becomes unavailable, the client automatically connects to the other.</p>
<figure class="wp-caption aligncenter" id="attachment_44074" style="width: 500px;"><img alt="Figure 3" class="size-full wp-image-44074" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-3.png" width="500" /><figcaption class="wp-caption-text" id="caption-attachment-44074">Fig 3: In active-active mode, both regions accept session writes while WAN replication keeps data synchronized.</figcaption></figure>
<p>With both clusters running (using <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">docker-compose up</span>), we can start the application, add items to a basket, and observe the session entries appearing in both clusters via Hazelcast Management Center:</p>
<p><img alt="Figure 4" class="size-full wp-image-44075" height="1000" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-4.png" width="1600" /></p>
<p>Now, let&#8217;s simulate a regional outage by shutting down the London cluster:</p>
<pre class="EnlighterJSRAW">docker-compose down london-cluster</pre>
<p>The application continues working without interruption. The Hazelcast failover client automatically connects to the remaining NYC cluster, and users can keep adding items to their basket without losing their session.</p>
<p><img alt="Figure 5" class="size-full wp-image-44077" height="515" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-5.png" width="1600" /></p>
<p>Next, we bring London back: </p>
<pre class="EnlighterJSRAW">docker-compose up london-cluster</pre>
<p>After a few seconds, WAN replication re-synchronizes the session data between clusters. Looking at the Management Center again, we can see the restored entries, confirming both regions are once again fully aligned.</p>
<p><img alt="Figure 6" class="size-full wp-image-44078" height="991" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-6.png" width="1600" /></p>
<h2 class="h3">From local sessions to global resilience</h2>
<p>With a small amount of configuration and no changes to your application code, we turned a plain <span style="font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">HttpSession</span> into a distributed, fault-tolerant system.</p>
<p>Sessions are no longer tied to a single server.</p>
<p>They survive instance restarts.</p>
<p>They can replicate across regions.</p>
<p>They can even run active-active with Hazelcast Enterprise.</p>
<p>And the application code stays exactly the same.</p>
<p>What starts as a simple dependency and annotation can evolve into a production-ready, multi-region session architecture. Even if there’s a complete blackout in the server room, your users can still come back and finish buying tons of cheap-but-not-necessary stuff on Black Friday!</p>
<p>Resilient user sessions are easier than you think!</p>
<figure class="wp-caption aligncenter" id="attachment_44079" style="width: 500px;"><img alt="Figure 7" class="size-full wp-image-44079" src="https://hazelcast.com/wp-content/uploads/2026/02/fig-7.png" width="500" /><figcaption class="wp-caption-text" id="caption-attachment-44079">Fig 7: The evolution of session resilience with Hazelcast Spring Session: Taking you from local sessions to fully resilient, multi-region session storage.</figcaption></figure>
<h2 class="h3">How to get started</h2>
<p>For <b>the full example</b>, please visit our <a href="https://github.com/hazelcast/hazelcast-code-samples/tree/master/spring/resilient-sessions">code samples repository on GitHub</a></p>
<p>You can get <b>Hazelcast Open Source</b> from Maven Central or using <a href="https://docs.hazelcast.com/hazelcast/latest/getting-started/install-hazelcast">other supported ways</a>. </p>
<p>You can also try <b>Hazelcast Enterprise</b> for great features like WAN replication &#8211; get a trial license <a href="https://hazelcast.com/get-started/">here</a> and download the distribution from our <a href="https://hazelcast.com/get-started/download/">webpage</a> or install it in another<a href="https://docs.hazelcast.com/hazelcast/latest/getting-started/install-enterprise"> supported way</a>.</p>
<p><b>Hazelcast Spring Session</b> repository: <a href="https://github.com/hazelcast/hazelcast-spring-session">https://github.com/hazelcast/hazelcast-spring-session</a></p>
<p><b>Spring Session Core</b> repository: <a href="https://github.com/spring-projects/spring-session">https://github.com/spring-projects/spring-session</a> </p>
<p>The post <a href="https://hazelcast.com/blog/resilient-user-sessions-are-easier-than-you-think-with-hazelcast-spring-session/">Resilient user sessions are easier than you think with Hazelcast Spring Session</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
