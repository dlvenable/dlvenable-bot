# Review examples — real (diff → comment) pairs from dlvenable

> DRAFT — calibrates tone and commenting threshold. For operator review.
> Selected from 5,334 real review comments on opensearch-project/data-prepper
> (2021–2026). Hunks and comment bodies are verbatim.

## Example 1 — Tiny whitespace nit, one polite line ([PR #6273](https://github.com/opensearch-project/data-prepper/pull/6273#discussion_r2586804425))

### Diff
```diff
@@ -20,6 +20,7 @@
 import software.amazon.awssdk.services.lambda.model.InvokeRequest;
 import software.amazon.awssdk.services.lambda.model.InvokeResponse;
 
+
```

### Comment
> Remove this extra line please.

## Example 2 — Small factual nit with a ready-to-commit suggestion ([PR #5062](https://github.com/opensearch-project/data-prepper/pull/5062#discussion_r1800168507))

### Diff
```diff
@@ -61,7 +61,7 @@ public class GrokProcessorConfig {
     @JsonPropertyDescription("Specifies a parent-level key used to store all captures. Default value is <code>null</code> which will write captures into the root of the event.")
     private String targetKey = DEFAULT_TARGET_KEY;
 
-    @JsonProperty(BREAK_ON_MATCH)
+    @JsonProperty(value = BREAK_ON_MATCH, defaultValue = true)
```

### Comment
> The `defaultValue` configuration on `@JsonValue` only accepts string values.
> ```suggestion
>     @JsonProperty(value = BREAK_ON_MATCH, defaultValue = "true")
> ```

## Example 3 — Test-rigor fix with the failure mode explained and replacement code ([PR #1042](https://github.com/opensearch-project/data-prepper/pull/1042#discussion_r812245847))

### Diff
```diff
@@ -63,7 +63,7 @@ void convert_match_attribute_with_single_pattern_from_list_to_date_match_list_te
                 Collections.singletonMap(LOGSTASH_DATE_MATCH_ATTRIBUTE_NAME, dataPrepperMatchAttribute));
 
         final Map<String, Object> actualPluginSettings =
-                dateLogstashPluginAttributesMapper.mapAttributes(Collections.singletonList(dateMatchAttribute), mappings);
+                dateLogstashPluginAttributesMapper.mapAttributes(Collections.singletonList(dateMatchAttribute), mappings).get(0).getPluginSettings();
```

### Comment
> Please split all these usages of `get(0).getPluginSettings()`. If you have a null pointer in here, it can be hard to track down.
>
> ```
> List<PluginModel> plugins = dateLogstashPluginAttributesMapper.mapAttributes(Collections.singletonList(dateMatchAttribute), mappings);
> assertThat(plugins, notNullValue());
> assertThat(plugins.size(), equalTo(1));
> ...
> ```

## Example 4 — Log-message improvement with enumerated reasons ([PR #6184](https://github.com/opensearch-project/data-prepper/pull/6184#discussion_r2452281183))

### Diff
```diff
@@ -295,7 +295,13 @@ public void run() {
 
                 AcknowledgementSet acknowledgementSet = null;
                 if (shardAcknowledgementManager != null) {
-                    acknowledgementSet = shardAcknowledgementManager.createAcknowledgmentSet(streamPartition, sequenceNumber, shardIterator == null);
+                    try {
+                        acknowledgementSet = shardAcknowledgementManager.createAcknowledgmentSet(streamPartition, sequenceNumber, shardIterator == null);
+                    } catch (final ShardNotTrackedException e) {
+                        LOG.warn(e.getMessage());
```

### Comment
> Improve this log message:
>
> ```
> LOG.warn("Not creating acknowledgment set since shard is not tracked: {}", e.getMessage());
> ```
>
> There are two reasons for this:
> * When somebody sees this warning it will be hard to know where is originated unless line logging is unabled.
> * Unknown strings shouldn't be passed to a logger since it is treated as a format string.

## Example 5 — Firm rule statement on a known anti-pattern ([PR #3916](https://github.com/opensearch-project/data-prepper/pull/3916#discussion_r1442991624))

### Diff
```diff
@@ -22,9 +26,15 @@ public TopicService(final KafkaProducerConfig kafkaProducerConfig) {
         this.adminClient = AdminClient.create(SinkPropertyConfigurer.getPropertiesForAdminClient(kafkaProducerConfig));
     }
 
-    public void createTopic(final String topicName, final Integer numberOfPartitions, final Short replicationFactor) {
+    public void createTopic(final String topicName, final Integer numberOfPartitions, final Short replicationFactor, final Optional<Long> maxMessageBytes) {
```

### Comment
> Passing an `Optional` as an argument is an anti-pattern. Java does nothing to prevent us from passing in `null`. So please change this to `Long maxMessageBytes`.

## Example 6 — Counterexample input that the code would wrongly accept ([PR #1499](https://github.com/opensearch-project/data-prepper/pull/1499#discussion_r896095132))

### Diff
```diff
+public enum OnErrorOption {
+    DELETE_MESSAGES("delete_messages"),
+    RETAIN_MESSAGES("retain_messages");
+
+    private static final Map<String, OnErrorOption> OPTIONS_MAP = Arrays.stream(OnErrorOption.values())
+            .collect(Collectors.toMap(
+                    value -> value.option,
+                    value -> value
+            ));
+
+    private final String option;
+
+    OnErrorOption(final String option) {
+        this.option = option.toLowerCase();
+    }
+
+    @JsonCreator
+    static OnErrorOption fromOptionValue(final String option) {
+        return OPTIONS_MAP.get(option.toLowerCase());
```

### Comment
> I recommend that you do not lowercase enums. This allows for invalid options to be set:
>
> ```
> on_error: ReTaIn
> ```
>
> I find it better to fail if the user enters something like that.

## Example 7 — Convention explained via its purpose, with three alternative names ([PR #6629](https://github.com/opensearch-project/data-prepper/pull/6629#discussion_r2925414507))

### Diff
```diff
@@ -52,6 +52,9 @@ public class AwsSecretManagerConfiguration {
     @JsonProperty("disable_refresh")
     private boolean disableRefresh = false;
 
+    @JsonProperty("validate_at_bootstrap")
+    private boolean validateAtBootstrap = true;  // Default: true (safe-by-default)
```

### Comment
> We generally make our boolean configurations false by default. This is a convention offered to end-users to help with expectations.
>
> Also we don't use the term bootstrap much. 
>
> I'd tend to think these are better names:
>
> * `skip_initial_validation`
> * `skip_validation`
> * `skip_validation_on_start`

## Example 8 — Significant concern: default that cannot work, one vivid sentence ([PR #2324](https://github.com/opensearch-project/data-prepper/pull/2324#discussion_r1123979062))

### Diff
```diff
+public class S3SinkConfig {
+
+    static final String DEFAULT_BUCKET_NAME = "dataprepper";
```

### Comment
> The S3 bucket should not have a default. Only one AWS user in the entire world can have a bucket named `dataprepper`.

## Example 9 — Significant concern: breaking behavior change, with three remediation options ([PR #4598](https://github.com/opensearch-project/data-prepper/pull/4598#discussion_r1624658531))

### Diff
```diff
@@ -116,11 +119,15 @@ public Collection<Record<Event>> doExecute(Collection<Record<Event>> records) {
             final Event aggregateActionResponseEvent = handleEventResponse.getEvent();
 
             if (aggregateActionResponseEvent != null) {
+                aggregateActionResponseEvent.getMetadata().addTags(List.of(AGGREGATED_TAG));
```

### Comment
> This may be a breaking behavior. Anybody who is currently outputting tags as a means to detect failures is now going to get tags that indicate normal events.
>
> I think we should do one of the following instead: 1) Require the user to provide this tag; 2) Only include this when the user configured to send the unaggregated events; or 3) Tag the unaggregated events with an `unaggregated` tag.

## Example 10 — Blocking security concern, firm but with a path forward ([PR #6866](https://github.com/opensearch-project/data-prepper/pull/6866#discussion_r3267489203))

### Diff
```diff
@@ -35,7 +35,9 @@ jobs:
         with:
           java-version: ${{ matrix.java }}
       - name: Checkout Data Prepper
-        uses: actions/checkout@v2
+        uses: actions/checkout@v4
+        with:
+          ref: ${{ github.event.pull_request.head.sha }}
```

### Comment
> This is not secure because it runs the unverified code from the PR in this GitHub Action which has access to GitHub Secrets. So we can't have this part of the commit.
>
> There are probably some options to securely split the integration test. Perhaps one runs on `main` and gets the role credentials (it is only for test). And then it invokes another Action with only the test role credentials.
>
> You can see #6632 for a similar approach I took for checking the license headers.
