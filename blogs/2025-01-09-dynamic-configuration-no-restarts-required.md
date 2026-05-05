---
title: "Dynamic Configuration: No Restarts Required"
url: "https://hazelcast.com/blog/dynamic-configuration-no-restarts-required/"
date: "Thu, 09 Jan 2025 14:00:41 +0000"
author: "Emre Yigit"
feed_url: "https://hazelcast.com/feed/"
---
<p><span style="font-weight: 400;">Hazelcast Platform is known for its high-performance data caching and integration capabilities and provides a robust and scalable framework for distributed applications. The platform offers numerous ways to</span><a href="https://docs.hazelcast.com/hazelcast/latest/configuration/understanding-configuration"> <span style="font-weight: 400;">configure clusters</span></a><span style="font-weight: 400;">—from XML or YAML files to environment variables, system properties, and programmatic APIs.</span></p>
<p><span style="font-weight: 400;">But what if your configuration needs to evolve while your Hazelcast Platform cluster is live? Enter </span><b>dynamic configuration</b><span style="font-weight: 400;">. It enables you to add new data structure configurations at runtime </span><i><span style="font-weight: 400;">without</span></i><span style="font-weight: 400;"> restarting the cluster, ensuring uninterrupted service even in production environments.</span></p>
<h3><b>Why dynamic configuration?</b></h3>
<p><span style="font-weight: 400;">Traditional methods of reconfiguring a cluster often necessitate downtime—a costly and disruptive process, especially in production. Dynamic configuration solves this by enabling the following:</span></p>
<ul>
<li style="font-weight: 400;"><span style="font-weight: 400;">Runtime updates to configuration, such as adding new data structures</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Persistence of changes, ensuring they survive cluster restarts</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">A flexible, programmatic API for seamless updates</span></li>
</ul>
<p><span style="font-weight: 400;">Dynamic configuration* is similar to a </span><a href="https://docs.hazelcast.com/hazelcast/latest/configuration/understanding-configuration"><span style="font-weight: 400;">static</span></a><span style="font-weight: 400;"> Hazelcast Platform configuration and can be called over similar interfaces. You can append your new data structure configuration to the config file or call programmatic APIs to set the new configuration. Dynamically added configuration can also be persisted, even when the cluster is restarted or shut down.</span></p>
<p><a href="https://docs.hazelcast.com/hazelcast/latest/configuration/dynamic-config-programmatic-api"><span style="font-weight: 400;">Hazelcast Platform documentation</span></a><span style="font-weight: 400;"> has more information on dynamic configuration.</span></p>
<h3><b>Use case: Configure a new distributed map using the API</b></h3>
<p><span style="font-weight: 400;">In the following scenario, you’re</span><a href="https://docs.hazelcast.com/hazelcast/latest/getting-started/get-started-binary"> <span style="font-weight: 400;">running a Hazelcast Platform cluster</span></a><span style="font-weight: 400;"> and must introduce a new map data structure with custom configurations </span><span style="font-weight: 400;">requiring</span><span style="font-weight: 400;"> disk persistence. Restarting the cluster to apply these changes isn’t an option in production, so dynamic configuration is employed.</span></p>
<h4><b>Connect to the cluster</b></h4>
<p><span style="font-weight: 400;">Start by initializing a Java client to connect to your cluster.</span></p>
<p><span style="font-weight: 400;">If you use a client other than Java, check the</span><a href="https://hazelcast.com/developers/clients/"> <span style="font-weight: 400;">feature list</span></a><span style="font-weight: 400;"> to ensure your client supports dynamic configuration. </span></p>
<pre class="EnlighterJSRAW">ClientConfig clientConfig = new ClientConfig();
clientConfig.setClusterName("dev"); 
clientConfig.getNetworkConfig().addAddress("127.0.0.1:5701");
HazelcastInstance client = HazelcastClient.newHazelcastClient(clientConfig );</pre>
<p><span style="font-weight: 400;">So far, the config is similar to a static one. You can interact with the cluster using the client.</span></p>
<pre class="EnlighterJSRAW">IMap&lt;String, String&gt; myFirstMap = client.getMap("myFirstMap");
myFirstMap.put("myKey", "myValue");</pre>
<p><span style="font-weight: 400;">“</span><i><span style="font-weight: 400;">myFirstMap</span></i><span style="font-weight: 400;">” is initialized with default map configuration values since no configuration was applied when starting the cluster, as per the initial requirements.</span></p>
<p><span style="font-weight: 400;">However, we now need a map structure that persists data to the disk. This will require a new map configuration because data persistence is not enabled by default. Since this is a production environment, dynamic configuration can be used to apply the new configuration and avoid restarting the cluster.</span></p>
<h4><b>Create a new configuration with persistence enabled</b></h4>
<p><span style="font-weight: 400;">Next, you will create a new configuration with persistence enabled and set the new map configuration to the cluster via the client.</span></p>
<pre class="EnlighterJSRAW">// Let's create the map config MapConfig mapConfig = new MapConfig();
mapConfig.setName("my-persistent-map") .setDataPersistenceConfig(new DataPersistenceConfig().setEnabled(true));
// Then, set the new config over the client client.getConfig().addMapConfig(mapConfig);
</pre>
<p><span style="font-weight: 400;">Now, the “my-persistent-map” map will persist to the disk and can be used immediately without restarting the cluster.</span></p>
<p><span style="font-weight: 400;">The programmatic APIs were used in the above scenario since there were no constraints on deploying Java code. However, if introducing new code is a constraint, we need another method of using dynamic configuration.</span></p>
<h3><b>Use case: Configure a new map using a config file</b></h3>
<p><span style="font-weight: 400;">If you can’t modify the cluster with the programmatic APIs, you can update your configuration file and dynamically reload it.</span></p>
<p><span style="font-weight: 400;">In this scenario, the new map configuration is appended to the static config file, typically `hazelcast.yaml` or hazelcast.xml. In our example, it&#8217;s the `hazelcast.xml` file.</span></p>
<pre class="EnlighterJSRAW">&lt;hazelcast&gt; ... 
&lt;map name="my-persistent-map"&gt; &lt;data-persistence enabled="true"&gt; &lt;fsync&gt;false&lt;/fsync&gt; &lt;/data-persistence&gt; &lt;/map&gt;
&lt;/hazelcast&gt; </pre>
<p><span style="font-weight: 400;">The cluster is now ready to consume the new map configuration.  </span></p>
<p><span style="font-weight: 400;">Reload the updated configuration via the Hazelcast Management Center or REST API, avoiding downtime entirely.</span></p>
<p><b>Tip!</b><span style="font-weight: 400;"> Management Center also accepts a configuration string to propagate to the cluster. If you can’t edit the config file, use this feature instead. </span></p>
<p><span style="font-weight: 400;">Also, our </span><a href="https://docs.hazelcast.com/hazelcast/latest/maintain-cluster/dynamic-config-via-rest#step-4-optional-dynamically-add-new-map-by-reloading-configuration-from-disk."><span style="font-weight: 400;">documentation has more information</span></a><span style="font-weight: 400;"> on configuring dynamic configuration over REST.</span><a href="https://docs.hazelcast.com/hazelcast/latest/maintain-cluster/dynamic-config-via-rest#step-4-optional-dynamically-add-new-map-by-reloading-configuration-from-disk"><span style="font-weight: 400;"> </span></a></p>
<h4><b>Handling split-brain scenarios</b></h4>
<p><span style="font-weight: 400;">Split-brain syndrome, or network partitioning, occurs in distributed computing systems when a network failure causes the system to split into two or more isolated groups of nodes (or servers). Unaware of the other, each group may attempt to operate independently, often leading to inconsistencies, duplicate actions, or conflicts (</span><a href="https://docs.hazelcast.com/hazelcast/latest/network-partitioning/network-partitioning#split-brain-syndrome"><span style="font-weight: 400;">more information</span></a><span style="font-weight: 400;">).</span></p>
<p><span style="font-weight: 400;">If you add a new configuration during such an occurrence, the other parts of the cluster will be unaware of it. However, during recovery, the merging side adopts the dynamically added configuration from the initiating segment. The merging ensures consistency once the cluster is whole again.</span></p>
<h3><b>Looking ahead</b></h3>
<p><span style="font-weight: 400;">Dynamic configuration supports adding new data structures but </span><i><span style="font-weight: 400;">not</span></i><span style="font-weight: 400;"> modifying existing ones at runtime. Hazelcast is constantly expanding its dynamic configuration capabilities and is considering developing the following new features to apply changes at runtime on an existing distributed map. In addition to persisting dynamic config changes, the new features would dynamically update:</span></p>
<ul>
<li style="font-weight: 400;"><span style="font-weight: 400;">Map size limits — set size limits on maps to prevent them from consuming too much memory  </span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Max size policy — determine how the maximum size of a map or Near Cache is measured and enforced</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Eviction policy — determine which entries should be removed from a map or cache when it reaches its maximum size limit</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Time to Live (TTL) — limit the lifetime of entries in a map</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Read from backup (read-backup-data) — </span><span style="font-weight: 400;">allow local members to read map data from their local backups when using Hazelcast in embedded mode</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">Max idle — limit the lifetime of entries in a map based on their last access time</span></li>
<li style="font-weight: 400;"><span style="font-weight: 400;">WAN replication references — enable WAN replication for specific data structures like maps and caches</span></li>
</ul>
<p><span style="font-weight: 400;">You can explore Hazelcast Platform&#8217;s dynamic configuration capabilities in the <a href="https://hazelcast.com/community-edition-projects/downloads/">Community Edition available here.</a></span></p>
<p><span style="font-weight: 400;">Join our Community Slack channel to shape future features and stay connected with Hazelcast.</span></p>
<p>The post <a href="https://hazelcast.com/blog/dynamic-configuration-no-restarts-required/">Dynamic Configuration: No Restarts Required</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
