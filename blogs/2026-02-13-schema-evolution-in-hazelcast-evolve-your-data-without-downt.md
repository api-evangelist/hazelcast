---
title: "Schema Evolution in Hazelcast: Evolve your data without downtime"
url: "https://hazelcast.com/blog/schema-evolution-in-hazelcast-evolve-your-data-without-downtime/"
date: "Fri, 13 Feb 2026 20:28:18 +0000"
author: "Fabrizio Cannizzo"
feed_url: "https://hazelcast.com/feed/"
---
<p>Applications change continuously: business rules evolve, new requirements appear, and data models change to accommodate these changes. In systems that serve critical workloads, those changes must happen while the system is live.</p>
<p>With a shared data layer like Hazelcast, you often end up with multiple application versions running at once, all reading and writing the same data but with different object layouts. Without explicit support for schema evolution, old code can’t read new data, and upgrades may end up in downtime.</p>
<p>Hazelcast solves this with <a href="https://docs.hazelcast.com/hazelcast/5.6/serialization/compact-serialization">Compact Serialization</a> and its support for <a href="https://docs.hazelcast.com/hazelcast/5.6/serialization/compact-serialization#schema-evolution">schema evolution</a>. In this post, I’ll walk through how this works in practice, what changes are safe, which ones are not, and how to evolve live data resiliently.</p>
<p>The internals of Compact Serialization (binary layout, schema fingerprint generation, replication, and partial deserialization) are covered in <a href="https://hazelcast.com/blog/compact-serialization-in-depth/">this blog post</a>, so I won’t repeat them here.</p>
<p>Many thanks to <a href="https://github.com/k-jamroz">Krzysztof Jamróz</a> for helping clarify edge cases around schema evolution and Compact Serialization and for reviewing this post.</p>
<h2 class="h3">Hazelcast support for schema evolution</h2>
<p>With <a href="https://docs.hazelcast.com/hazelcast/latest/serialization/compact-serialization">Compact Serialization,</a> Hazelcast separates an object’s schema (its field names and types) from the binary data stored in memory or on disk.</p>
<p>Each distinct schema is identified by a 64-bit fingerprint derived from its structure, and each record references the fingerprint of the schema used when it was written. This allows multiple schema versions of the same logical type to coexist in the cluster. Schemas are registered when first encountered and exchanged between members and clients on demand, and when persistence is enabled, they are stored alongside the data to allow exact recovery after a restart.</p>
<h3 class="h4">Key Terms</h3>
<ul>
<li><strong>Schema</strong>: the structural definition of a serialized object, including its field names and types</li>
<li><strong>Type name</strong> (typeName): a unique and stable identifier associated with the Compact type. It binds all schema versions for a given logical class</li>
<li><strong>Fingerprint</strong>: a 64-bit identifier derived from the schema definition. Hazelcast uses a <a href="https://en.wikipedia.org/wiki/Rabin_fingerprint">Rabin Fingerprint</a> algorithm to determine the identifier, and every time the schema changes, a new fingerprint is generated.</li>
</ul>
<p>Hazelcast uses the typeName to decide <em>what</em> an object represents, and the fingerprint to determine which<em> version</em> of the schema was used.</p>
<h2 class="h3">Compatible vs incompatible changes</h2>
<p>When a schema changes, compatibility is crucial because, during rolling upgrades of the application, both old and new versions may run concurrently, reading and writing the same data through a shared Hazelcast cluster.</p>
<p>From Hazelcast’s point of view, a change is compatible if existing readers can still deserialize data written with the new schema. From the application’s point of view, it is only compatible if the data still makes sense to the business logic. In practice, compatible changes allow old and new schema versions to coexist during application rolling upgrades against a stable Hazelcast cluster, while incompatible changes require explicit migration to transform existing data.</p>
<p>As a rule of thumb, compatible changes include adding optional fields, removing fields that readers no longer depend on, widening numeric types when readers handle both representations, and extending nested objects in an additive way. These changes can usually be introduced without rewriting existing data.</p>
<p>Incompatible changes include narrowing types, renaming fields, changing field meaning, restructuring nested types, switching between unrelated type families, changing date or timestamp semantics (such as time zones or epoch units), and altering partitioning or key structure. These changes require explicit migration, because schema evolution alone does not address semantic correctness.</p>
<p>The key distinction is that compatibility here refers to serialization, not meaning. Date and time changes are a common case where deserialization succeeds but semantics change, so migrations should always be explicit and verified.</p>
<p>More details on compatible vs incompatible changes are available in the <a href="https://docs.hazelcast.com/hazelcast/5.6/serialization/schema-evolution-full#decision-table">Hazelcast’s official documentation</a>.</p>
<h2 class="h3">A simple Compact example</h2>
<p>Full code in support of this blog is available on GitHub in the <a href="https://github.com/fcannizzohz/schema-migration-sample">schema-migration-samples</a> repo.</p>
<p>To show how to handle changes in practice, we’ll follow up with a concrete example. Let’s start with a basic <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Order</span> class:</p>
<pre class="EnlighterJSRAW">public record Order(long id, long customerId, BigDecimal amount, String status) {}</pre>
<p>and the respective compact <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderSerializer</span>:</p>
<pre class="EnlighterJSRAW">public final class OrderSerializer implements CompactSerializer&lt;Order&gt; {

  @Override
  public String getTypeName() {
      return "com.acme.Order";
  }
  
  @Override
  public Class&lt;Order&gt; getCompactClass() {
      return Order.class;
  }
  
  @Override
  public void write(CompactWriter w, Order o) {
      w.writeInt64("id", o.id());
      w.writeInt64("customerId", o.customerId());
      w.writeDecimal("amount", o.amount());
      w.writeString("status", o.status());
  }
  
  @Override
  public Order read(CompactReader r) {
      return new Order(
          r.readInt64("id"),
          r.readInt64("customerId"),
          r.readDecimal("amount"),
          r.readString("status")
      );
  }
}
</pre>
<h2 class="h3">Strategies to handle compatible schema evolution</h2>
<p>Using the previous example as a starting point, we can now look at practical strategies for handling schema evolution. For data that is cached, applying these strategies means using the same map to handle objects that have different compatible schema changes.</p>
<h3 class="h4">Keeping <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span> stable for compatible changes</h3>
<p>In Compact Serialization, the <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span> acts as the logical identity of a type. Hazelcast uses it to decide whether two schemas describe the same concept or represent entirely different data.</p>
<p>As long as schema changes are compatible, different schema versions can safely share the same <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span>. In that case, Hazelcast treats them as evolutions of the same type, allowing old and new application code to exchange data transparently. When the <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span> changes, that continuity is deliberately broken: Hazelcast no longer attempts to relate the new schema to the old one, and the data is treated as belonging to a different type.</p>
<p>This distinction becomes important during rolling application upgrades, where multiple schema versions may coexist in the cluster at the same time.</p>
<h3 class="h4">Deterministic writes</h3>
<p>Compact schema fingerprints are derived from the fields written by the serializer. If the set of written fields changes depending on runtime conditions or object state, Hazelcast may observe multiple schemas for what is logically the same type.</p>
<p>This is where problems start to appear. Multiple fingerprints for the same logical structure make schema evolution unpredictable and can lead to unnecessary schema registrations or deserialization failures. In practice, serializers that always write the same fields in the same order produce stable fingerprints and predictable evolution behavior.</p>
<h2 class="h3">Safely evolve <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Order</span></h2>
<p>Let’s see how to apply the strategies described above.</p>
<p>Let’s introduce a new <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderV2</span> class that adds a new field, “<span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">currency</span>” to the schema. This is a compatible change that allows <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Order</span> and <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderV2</span> objects to be serialzied in the same map (e.g. <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">orders</span>).</p>
<pre class="EnlighterJSRAW">public record OrderV2(long id, long customerId, BigDecimal amount, String status, String currency) {}

public final class OrderV2Serializer implements CompactSerializer&lt;OrderV2&gt; {
    @Override
    public String getTypeName() {
        return "com.acme.Order"; // same typeName: same logical type
    }

    @Override
    public Class&lt;OrderV2&gt; getCompactClass() {
        return OrderV2.class;
    }
    
    @Override
    public void write(CompactWriter w, OrderV2 o) {
        w.writeInt64("id", o.id());
        w.writeInt64("customerId", o.customerId());
        w.writeDecimal("amount", o.amount());
        w.writeString("status", o.status());
        w.writeString("currency", o.currency());
    }

  @Override
  public OrderV2 read(CompactReader r) {
      String currency = "GBP";
      if (r.getFieldKind("currency") == FieldKind.STRING) {
          currency = Optional.ofNullable(r.readString("currency")).orElse("GBP");
      }
      return new OrderV2(
          r.readInt64("id"),
          r.readInt64("customerId"),
          r.readDecimal("amount"),
          r.readString("status"),
          currency
      );
  }
}
</pre>
<p>Both serializers share the same <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span> and deterministic field order. <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderV2Serializer</span> remains backward compatible by providing a default for <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">currency</span> when the field is missing.</p>
<p>As a result:</p>
<ul>
<li>When <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderSerializer</span> reads an OrderV2 object, <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">currency</span> is ignored. This applies to full Compact deserialization paths; SQL and index evaluation operate directly on stored fields.</li>
<li>When <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderV2Serializer</span> reads an <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Order</span> object, <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">currency</span> is defaulted.</li>
</ul>
<p>Backward and forward compatibility are maintained.</p>
<h2 class="h3">When a change breaks compatibility</h2>
<p>As discussed above, some changes break old readers that cannot interpret the data correctly.</p>
<p>Once this compatibility is broken, coexistence is no longer sufficient. Schema evolution alone cannot bridge the gap, because the structure or semantics of the data no longer match what the application expects. At that point, existing data must be transformed to align with the new schema and its meaning.</p>
<p>This is why incompatible changes require explicit migration. Migration makes the transition intentional by converting data written under the old schema into a form that the new version can consume safely, restoring correctness before normal operation resumes.</p>
<p>There are several ways to implement migration once compatibility is broken.</p>
<p>In some cases, applications embed an explicit version field and handle multiple representations in the reader logic. In others, data is migrated explicitly, and old and new representations are kept separate during the transition. Here, we focus on a simple and explicit approach based on using multiple map versions, which makes schema boundaries clear and keeps migration logic isolated from normal read and write paths.</p>
<p>Let’s create a new schema for Order that introduces a breaking change by renaming the field <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">customerId</span> to <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">accountId</span></p>
<pre class="EnlighterJSRAW">public record OrderV3(long id, long accountId, BigDecimal amount, String status, String currency) {}</pre>
<p>The new serializer is:</p>
<pre class="EnlighterJSRAW">public final class OrderV3Serializer implements CompactSerializer&lt;OrderV3&gt; {

    @Override
    public String getTypeName() {
        return "com.acme.OrderV3"; // new typeName: new schema
    }
  
    @Override
    public Class&lt;OrderV3&gt; getCompactClass() {
        return OrderV3.class;
    }
  
    @Override
    public void write(CompactWriter w, OrderV3 o) {
        w.writeInt64("id", o.id());
        w.writeInt64("accountId", o.accountId());
        w.writeDecimal("amount", o.amount());
        w.writeString("status", o.status());
        w.writeString("currency", o.currency());
    }
    
    @Override
    public OrderV3 read(CompactReader r) {
        return new OrderV3(
            r.readInt64("id"),
            r.readInt64("accountId"),
            r.readDecimal("amount"),
            r.readString("status"),
            r.readString("currency")
        );
    }
}
</pre>
<p>By introducing a new <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">typeName</span> (<span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">com.acme.OrderV3</span>), the new schema is explicitly separated from earlier versions. This makes it clear that the data represents a different structure and avoids ambiguity during reads and writes.</p>
<p>Data written under the old schema can no longer be used directly and must be transformed to match the new representation. Keeping old and new data separate during this transition makes the change easier to reason about and validate.</p>
<p>Because each map maintains its own Compact schemas, the two representations remain isolated and recoverable throughout the migration. In the following sections, we use Hazelcast Jet as simple examples of how this transformation can be performed.</p>
<h3 class="h4">Migration pipeline example</h3>
<p>For simplicity, assume that the <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">orders</span> map currently contains only <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">OrderV2</span> records. Migrating existing data then becomes a batch operation: entries are read from the old representation, transformed, and written using the new schema.</p>
<p>A Jet pipeline provides a concise way to express this transformation and runs to completion once all existing records have been rewritten.</p>
<pre class="EnlighterJSRAW">Pipeline bulk = Pipeline.create();

bulk.readFrom(Sources.&lt;Long, OrderV2&gt;map("orders"))
    .map(e -&gt; Map.entry(
        e.getValue().id(),
        new OrderV3(/* transformed value */)
    ))
    .writeTo(Sinks.map("orders_v3"));

hz.getJet().newJob(bulk).join();</pre>
<p>In practice, migrations often run alongside a rolling application upgrade. While the batch job is processing existing entries, older versions of the application may still be writing to the original map.</p>
<p>To keep the new representation consistent during this transition, a second pipeline can consume changes from the source map’s event journal and apply them as they occur.</p>
<pre class="EnlighterJSRAW">Pipeline tail = Pipeline.create();

tail.readFrom(
        Sources.&lt;Long, OrderV2&gt;mapJournal(
            "orders",
            JournalInitialPosition.START_FROM_CURRENT
        )
    )
    .map(e -&gt; {
        if (e.getValue() != null) {
            return Map.entry(
                e.getValue().id(),
                new OrderV3(/* transformed value */)
            );
        } else {
            return null; // represents a delete
        }
    })
    .writeTo(Sinks.map("orders_v3"));

hz.getJet().newJob(tail);</pre>
<p>The batch pipeline handles the existing dataset, while the journal-driven pipeline keeps the new schema up to date by propagating inserts, updates, and deletes as they happen. Together, these pipelines allow data to be migrated incrementally without stopping the application until all clients have transitioned to the new schema.</p>
<p>The tail pipeline in the above code snippet is logically correct but incomplete. The full pipeline is available <a href="https://github.com/fcannizzohz/schema-migration-sample">on github</a> (<span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">com.fcannizzohz.samples.schemaevolution. migration.V2toV3PipelineFactory#createTailPipeline</span>).</p>
<h2 class="h3">Persistence and <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">MapStore</span></h2>
<p>If you use Persistence or <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">MapStore</span>, Compact schemas are saved alongside map data.<br />
Each record stores its schema fingerprint, so Hazelcast can restore it exactly as it was after a restart.</p>
<p>So the strategies described above carry on working as expected.</p>
<p>You must pay attention when both the MapStore and Jet pipeline operate on the same target map to handle backward incompatible changes and migration is required.</p>
<p>Since both MapStore and the job may operate on the same key, the order of operations determines which value persists.</p>
<p>If a key is not in memory, Hazelcast may call the MapStore’s <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">load</span> or <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">loadAll</span> methods to fetch it from the external system. If the load executes concurrently with the pipeline on the same key the order matters and the last value wins.</p>
<p>The Jet pipeline may be coded using <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Sinks.map</span>, <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Sinks.mapWithMerging</span>, <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Sinks.mapWithUpdating</span>, or <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">Sinks.mapWithEntryProcessor</span> to decide what strategy to adopt in case the pipeline attempts to sink a key that already exist.</p>
<h2 class="h3">Near cache behaviour</h2>
<p>When <strong>Near Cache</strong> is enabled, it remains unaffected by Compact schema evolution.<br />
Each map’s Near Cache stores its own serialized entries. Old clients use the old schema and cache; new clients use the new one. No invalidation is required.</p>
<h2 class="h3">SQL and Indexes</h2>
<p>When Compact-serialized data is queried through Hazelcast SQL, fields are read directly from the stored binary using the schema associated with each record. SQL does not invoke the application’s <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">CompactSerializer.read()</span> logic, so any defaulting or migration implemented there is not applied at query time.</p>
<p>This means that both old and new data can be queried using either the original mapping or an extended one, once the cluster is running a backward-compatible serializer. With the original mapping, queries continue to see only the fields defined in the old schema. With an extended mapping, newly added fields become queryable, but records written under earlier schemas simply return <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">NULL</span> for those columns, while records written with the new schema return actual values.</p>
<p>This behaviour reflects SQL’s schema-on-read model: data is not rewritten when schemas evolve, and each row is interpreted using the schema version it was written with.</p>
<p>In our orders example, with the following mapping, old schema orders (without the currency field) that match a SQL where clause are returned but with <span style="color: #000000; font-family: Source Code Pro, Liberation Mono, Courier New, Courier, monospace;">NULL</span> currency.</p>
<pre class="EnlighterJSRAW">CREATE OR REPLACE MAPPING orders (
    id BIGINT,
    customerId BIGINT,
    amount DECIMAL,
    status VARCHAR,
    currency VARCHAR
)
TYPE IMap
OPTIONS (
    'keyFormat' = 'bigint',
    'valueFormat' = 'compact',
    'valueCompactTypeName' = 'com.acme.Order'  -- matches Compact typeName in serializer
);</pre>
<p>Indexes follow the same boundaries as the data they describe. They are defined per map and remain valid across compatible schema changes.</p>
<p>New indexes can be added for newly introduced fields without affecting existing queries. When a schema change is incompatible and data is migrated explicitly, corresponding indexes must be recreated to match the new representation before queries rely on them.</p>
<h2 class="h3">Closing thoughts</h2>
<p>Compact Serialization turns schema evolution into a manageable part of running Hazelcast-powered systems, rather than something to work around.</p>
<p>Different schema versions can coexist in the same cluster, and compatibility rules are explicit instead of implicit. That’s what makes it possible to evolve applications without synchronized upgrades or planned downtime.</p>
<p>Most changes in a live system are additive: new fields, optional data, or reshaped objects that old code can safely ignore. If readers tolerate missing or additional fields, these changes can be rolled out while the system is running. When a change breaks compatibility, that break is visible and deliberate. You know a migration is required, instead of discovering it later through failed deserialization or corrupted state.</p>
<p>Handled this way, schema evolution stops being a special event. Upgrades carry less risk and become part of normal operations rather than something that disrupts them.</p>
<p>More details are available in the <a href="https://docs.hazelcast.com/hazelcast/5.5/serialization/compact-serialization#schema-evolution">Schema Evolution section</a> of the Hazelcast docs.</p>
<p><strong>Join our community and get started</strong></p>
<p>Stay engaged with Hazelcast—join our <a href="https://hazelcastcommunity.slack.com/join/shared_invite/zt-2vl6n8kfm-BmQjWkmzKQJtphAjR6m6qQ#/shared-invite/email">Community Slack channel</a> to connect with peers, share insights, and influence future features.  Sign up for a free <a href="https://hazelcast.com/get-started/">Enterprise Trial License</a> to try all our enterprise features. Build scalable, resilient systems today!</p>
<p>The post <a href="https://hazelcast.com/blog/schema-evolution-in-hazelcast-evolve-your-data-without-downtime/">Schema Evolution in Hazelcast: Evolve your data without downtime</a> appeared first on <a href="https://hazelcast.com">Hazelcast</a>.</p>
