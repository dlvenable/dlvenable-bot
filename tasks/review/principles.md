# Review principles — dlvenable (drafted from opensearch-project/data-prepper history)

> DRAFT distilled from 5,334 review comments (2021-09-14 to 2026-10-30) by the
> learn-review-principles skill on 2026-11-05. For operator review — edit freely.
>
> Scope notes:
> - Principles the target repo's own contributor docs already mandate are
>   excluded (the bot receives those docs separately). Principles below either
>   refine those docs or are absent from them.
> - Principles that only make sense in Data Prepper are labeled
>   **[repo-specific]**; everything else is written to apply to any Java repo
>   the bot reviews.
> - Sections are ordered by how often the theme recurred across the corpus.

## Tests must not be able to pass vacuously

- **Before a test loops over (or indexes into) a result collection, it must
  assert the collection's size; an empty collection otherwise makes every
  in-loop assertion a silent no-op.**
  Evidence: "Assert the length first. Or this loop doesn't test anything and passes." ([PR #6488](https://github.com/opensearch-project/data-prepper/pull/6488#discussion_r2814089197)), "Please assert the number of processors before this loop. Otherwise, this could result in no verification." ([PR #5127](https://github.com/opensearch-project/data-prepper/pull/5127#discussion_r1821397787))
- **Exception expectations use `assertThrows` with only the single call
  expected to throw inside the lambda — never a try-catch with assertions in
  the catch block, which silently skips all later verifications when nothing
  throws.**
  Evidence: "You should use `assertThrows()`. If this doesn't throw then all of your verify statements below will not run." ([PR #361](https://github.com/opensearch-project/data-prepper/pull/361#discussion_r723592415)), "It is best to reduce what is in the `assertThrows` so that you are sure you are testing the correct method fails." ([PR #435](https://github.com/opensearch-project/data-prepper/pull/435#discussion_r730039617))
- **Remove assertions that are tautologically true regardless of the code
  under test, and redundant null pre-checks that a later `assertThat` would
  already report.**
  Evidence: "This appears to be a tautology. Why is this test here?" ([PR #2444](https://github.com/opensearch-project/data-prepper/pull/2444#discussion_r1157323717))
- **Negative-path tests must explicitly verify the side-effecting call did NOT
  happen (`verify(mock, never())...`), not merely assert the happy-path result
  is absent.**
  Evidence: "I believe that we should verify that delete was not called here. `verify(sqsClient, never()).deleteMessageBatch(...)`" ([PR #1499](https://github.com/opensearch-project/data-prepper/pull/1499#discussion_r894869484))

## Test structure: one behavior per test, no control flow, no sleeps

- **Test methods must not contain if/else conditionals, scenario loops, or
  try-catch blocks. Split scenarios into separate tests or a
  `@ParameterizedTest`; add `throws Exception` to the signature instead of
  catching.**
  Evidence: "Let's not use conditionals in tests. The test become too complicated and we are not sure we are even testing what we want." ([PR #6507](https://github.com/opensearch-project/data-prepper/pull/6507#discussion_r2814208953)), "For tests, we should avoid try-catch statements. Just add `throws Exception` to the test signature." ([PR #3974](https://github.com/opensearch-project/data-prepper/pull/3974#discussion_r1457644580))
- **Near-duplicate test methods differing only in an input value become a
  JUnit 5 `@ParameterizedTest` (`@ValueSource`/`@CsvSource`/`@EnumSource`/
  ArgumentsProvider); boolean-input behavior is tested over `{true, false}`.**
  Evidence: "For boolean tests, we should use parameterized tests. It helps ensure that the code is not written to favor one boolean value over another." ([PR #756](https://github.com/opensearch-project/data-prepper/pull/756#discussion_r770890174)), "Use `@ParameterizedTest` with `@ValueSource(booleans = {true, false})` instead. These are two different test cases." ([PR #6596](https://github.com/opensearch-project/data-prepper/pull/6596#discussion_r2884368498))
- **No `Thread.sleep` or long fixed waits in tests: mock the collaborator,
  inject a `Clock`, mock `Thread.sleep` statically, or use Awaitility with a
  bounded condition.**
  Evidence: "We need to avoid long sleeps in our tests. They make the tests slow and result in flaky tests." ([PR #5095](https://github.com/opensearch-project/data-prepper/pull/5095#discussion_r1815511240)), "What are you waiting for in this time? Can we use awaitility instead to wait on a condition?" ([PR #6829](https://github.com/opensearch-project/data-prepper/pull/6829#discussion_r3227325874))
- **Construct the subject via a `createObjectUnderTest()` factory method
  called inside each test (not built in `@BeforeEach`), so individual tests
  can adjust setup before construction; group tests sharing distinct setup in
  `@Nested` classes.**
  Evidence: "We should use `createObjectUnderTest()` methods instead of creating in the setup method." ([PR #3437](https://github.com/opensearch-project/data-prepper/pull/3437#discussion_r1348170262))
- **When testing invalid configuration, vary exactly one invalid value while
  keeping all others valid, and test both boundary ends (values at and just
  beyond min/max, null, empty).**
  Evidence: "When testing invalid values, you should use all valid values except one specific invalid value. Otherwise, you don't know what you tested against." ([PR #6349](https://github.com/opensearch-project/data-prepper/pull/6349#discussion_r2620306942)), "I recommend also testing against values near `MAXIMUM_ALLOWED_VALUE` AND `MINIMUM_ALLOWED_VALUE`." ([PR #1405](https://github.com/opensearch-project/data-prepper/pull/1405#discussion_r878271140))

## Randomized test data — arbitrary values must look arbitrary

- **When a test value is arbitrary, generate it randomly
  (`UUID.randomUUID().toString()`, bounded random ints) instead of hardcoding
  literals, and assert against the generated value; never test with a value
  equal to the production default (a default leaking through would pass
  undetected).**
  Evidence: "It's better to use random strings. It helps clarify that the exact value doesn't matter." ([PR #2121](https://github.com/opensearch-project/data-prepper/pull/2121#discussion_r1080713653)), "You are setting a test value against the default constant. I recommend creating a random integer value, providing that in the constructor and then verifying against it." ([PR #473](https://github.com/opensearch-project/data-prepper/pull/473#discussion_r741157713))
- **Do not reuse the same literal for multiple fields in a test fixture —
  unique values per field prove each value flowed to the right place.**
  Evidence: "Can you update these tests to not repeat any values? ... It would be better to have them all unique to provide certainty that they are all working." ([PR #1042](https://github.com/opensearch-project/data-prepper/pull/1042#discussion_r808194903))

## Mockito and assertion precision

- **Tests that mock collaborators must verify the arguments actually passed —
  use `ArgumentCaptor` (or exact matchers like `eq(...)`) and assert on the
  captured values, not merely that a call happened or that nothing threw.**
  Evidence: "These tests should use `ArgumentCaptor<DeleteMessageBatchRequest>` to capture the actual argument and then inspect it." ([PR #1499](https://github.com/opensearch-project/data-prepper/pull/1499#discussion_r894813744)), "Rather than making this test oriented around this not throwing, we should be testing the desired behavior ... Use Mockito to verify this interaction" ([PR #6067](https://github.com/opensearch-project/data-prepper/pull/6067#discussion_r2331645438))
- **Prefer the matcher that reports actual-vs-expected on failure:
  `instanceOf()` over `assertTrue(x instanceof Y)`, `sameInstance()` when
  identity is the contract, `nullValue()` over `equalTo(null)`, `never()` over
  `times(0)`, `assertFalse(x)` over `assertTrue(!x)`; `times(1)` is the
  default and should be omitted. Hamcrest arguments are `(actual, matcher)`.**
  Evidence: "assertThat(result, instanceOf(EnumDeserializer.class)); This will output the class type if it fails the assertion." ([PR #5076](https://github.com/opensearch-project/data-prepper/pull/5076#discussion_r1811465054)), "I think you should use `sameInstance` to be very explicit ... This only works now because `equals`/`hashCode` is not yet implemented." ([PR #2290](https://github.com/opensearch-project/data-prepper/pull/2290#discussion_r1116099910))
- **No class-wide `lenient()` settings or unused stubs; keep stubbing outside
  `mockStatic` try-with-resources blocks and keep those blocks to the minimal
  lines that need the static mock; prefer Answer-based stubbing keyed on
  inputs over call-order-coupled `thenReturn` chains.**
  Evidence: "This will apply lenient to all mocks. Let's avoid this." ([PR #6256](https://github.com/opensearch-project/data-prepper/pull/6256#discussion_r2518792132)), "It would be better to use `whenAnswer`. ... This is creating a coupling of the order here with the order in the implementation that need not exist." ([PR #6910](https://github.com/opensearch-project/data-prepper/pull/6910#discussion_r3373786365))
- **Unit tests must not hit real networks/URLs/files when a mock suffices;
  integration tests are the inverse — no mocks, real objects, and never
  re-implement production logic to compute expected values (start from known
  input and verify real output).**
  Evidence: "Can we use mocking to avoid making real URL calls? ... They slow the tests down ... add volatility" ([PR #4002](https://github.com/opensearch-project/data-prepper/pull/4002#discussion_r1463526421)), "This is an integration test and should not be using mocks. Please create real objects." ([PR #2401](https://github.com/opensearch-project/data-prepper/pull/2401#discussion_r1157941332)), "You are duplicating the logic from `AvroInputCodec` in the test. This is not a valid test scenario." ([PR #2715](https://github.com/opensearch-project/data-prepper/pull/2715#discussion_r1200704613))
- **Config classes are tested by deserializing YAML/JSON with a Jackson
  `ObjectMapper` (or mocking the config class), not by reflectively setting
  private fields; where reflection is unavoidable use the shared test utility
  rather than hand-rolling it.** [repo-specific utility: `ReflectivelySetField`]
  Evidence: "The way to test the configs is to use a Jackson `ObjectMapper`." ([PR #2620](https://github.com/opensearch-project/data-prepper/pull/2620#discussion_r1191713171)), "We have a class named `ReflectivelySetField` in `common-test` that you can use." ([PR #5818](https://github.com/opensearch-project/data-prepper/pull/5818#discussion_r2241089303))

## Test coverage demands

- **A bug-fix PR must include a test that fails without the change and passes
  with it; new conditional branches and new methods with logic each need a
  targeted unit test.**
  Evidence: "Please add a unit tests that fails without this change but works with it. This way we know it works and it won't fail again if somebody changes the code." ([PR #6603](https://github.com/opensearch-project/data-prepper/pull/6603#discussion_r2913991620)), "Can you add a unit test which demonstrates the original failure and then shows that this fixes it?" ([PR #3998](https://github.com/opensearch-project/data-prepper/pull/3998#discussion_r1462485205))
- **Never lower an existing code-coverage threshold to make a build pass; new
  projects set the minimum to 100% and classes are not excluded from
  coverage.**
  Evidence: "We shouldn't lower this. Add unit tests as necessary." ([PR #4446](https://github.com/opensearch-project/data-prepper/pull/4446#discussion_r1575022618)), "We should be hitting 100% on new projects. Please set to `1.0`." ([PR #6627](https://github.com/opensearch-project/data-prepper/pull/6627#discussion_r2996806144))
- **Test public behavior, not private methods; if a test needs access to
  private members, that logic belongs in its own class with its own tests.**
  Evidence: "These tests are only testing a private method, not the actual behavior." ([PR #3510](https://github.com/opensearch-project/data-prepper/pull/3510#discussion_r1362210613))
- **[repo-specific] Every new plugin needs at least one test through the
  plugin framework (`@DataPrepperPluginTest` or an IT modeled on
  `GrokProcessorIT`) loading real YAML configuration.**
  Evidence: "We should have at least one test that uses the test plugin framework to verify behavior." ([PR #6229](https://github.com/opensearch-project/data-prepper/pull/6229#discussion_r2495274937))

## User-facing configuration conventions

- **Config keys are lowercase snake_case, full words only — no abbreviations
  (`number_retries` not `num_retries`), list-valued options are plural,
  file-path options end in `_file`, and no prefix that repeats the enclosing
  block or plugin name.**
  Evidence: "Please rename to `number_retries` to support our configuration consistency efforts." ([PR #2198](https://github.com/opensearch-project/data-prepper/pull/2198#discussion_r1090784197)), "The `azure_` prefix on these is redundant since it fits under the `azure_federated` config." ([PR #7011](https://github.com/opensearch-project/data-prepper/pull/7011))
- **Config names describe the user-facing concept and exactly what values are
  accepted — not the implementation or vendor SDK term (`socket_timeout` not
  `sdk_timeout`; `topic_arn` if only an ARN is accepted); reuse terminology
  already established in the project and in the upstream service's own API.
  When a config option maps to an upstream service parameter, look up that
  service's public API documentation (e.g. the AWS API Reference for the
  relevant operation) and align the option name with the official parameter
  name — do not guess from the SDK method or variable names in the diff.**
  Evidence: "Can this be a topic name? Or must it be an ARN? ... please name this `topic_arn`." ([PR #2995](https://github.com/opensearch-project/data-prepper/pull/2995#discussion_r1277777171)), "AWS SecretsManager uses the term `SecretId`, so let's name this `secret_id`." ([PR #3340](https://github.com/opensearch-project/data-prepper/pull/3340#discussion_r1327572544)), "The OpenSearch API uses the term `routing`, so I recommend that over `routing_id`." ([PR #3863](https://github.com/opensearch-project/data-prepper/pull/3863#discussion_r1439906754))
- **[repo-specific] Boolean config options default to false; if the desired
  default behavior is "on", invert and rename the flag (`disable_X`/`skip_X`)
  so false stays the default. Boolean fields use primitive `boolean`, never
  boxed `Boolean`.**
  Evidence: "We generally make our boolean configurations false by default. This is a convention offered to end-users to help with expectations." ([PR #6629](https://github.com/opensearch-project/data-prepper/pull/6629#discussion_r2925414507)), "Please use the primitive type `boolean` ... Using the primitive prevents it from being `null`." ([PR #5274](https://github.com/opensearch-project/data-prepper/pull/5274#discussion_r1900176131))
- **Options with a closed set of values must be a Java enum (not a free
  string), deserialized via `@JsonCreator`, with lowercase serialized values —
  and matching is case-sensitive: no `toLowerCase()`/`equalsIgnoreCase` on
  user input, and a lookup miss must fail, never silently fall back to a
  default.**
  Evidence: "I recommend that you do not lowercase enums. This allows for invalid options to be set: `on_error: ReTaIn`" ([PR #1499](https://github.com/opensearch-project/data-prepper/pull/1499#discussion_r896095132)), "Do not return a default value for `null` here." ([PR #5032](https://github.com/opensearch-project/data-prepper/pull/5032#discussion_r1819436591))
- **Time-valued options are `java.time.Duration` and byte-size options use a
  typed size value, with the unit suffix dropped from the name
  (`cache_ttl` not `cache_expiration_minutes`); keep the typed value as long
  as possible and convert at the point of use. Public method signatures take
  `Instant`/`Duration`, not epoch longs.** [repo-specific type: `ByteCount`]
  Evidence: "Please change this type to a `Duration`. Once you do that, Data Prepper core will parse this using a standard approach." ([PR #2324](https://github.com/opensearch-project/data-prepper/pull/2324#discussion_r1153497212)), "Rename to `s3_object_size_limit`. No need for `mb` when moving to `ByteCount`." ([PR #5992](https://github.com/opensearch-project/data-prepper/pull/5992#discussion_r2631365778))
- **Operational values users will need to vary (timeouts, thread counts,
  paths, batch sizes) must be configurable, and no option may ship a default
  that cannot universally work.**
  Evidence: "The S3 bucket should not have a default. Only one AWS user in the entire world can have a bucket named `dataprepper`." ([PR #2324](https://github.com/opensearch-project/data-prepper/pull/2324#discussion_r1123979062))
- **Avoid unnecessary YAML nesting: required options belong at the top level,
  and list items are not wrapped in single-key objects (use `List<X>` directly
  and delete the wrapper class).**
  Evidence: "We should try to avoid creating unnecessary nesting in the YAML." ([PR #2620](https://github.com/opensearch-project/data-prepper/pull/2620#discussion_r1191696829))

## Config validation is declarative and fails at startup

- **Validation lives in the config class as JSR-380/Jakarta annotations —
  `@NotNull` (in addition to `@NotEmpty`, which alone permits null), `@Min`
  plus a sane `@Max` upper bound, `@DurationMin`/`@DurationMax` — replacing
  hand-written `isValid()` methods and late runtime checks.**
  Evidence: "Java bean validation's `@NotEmpty` permits either `null` values or empty values ... Please add `@NotNull` as well." ([PR #1002](https://github.com/opensearch-project/data-prepper/pull/1002#discussion_r804117142)), "Please add an `@Min(1)` on this. We should probably also have some sort of upper bound ... Perhaps: `@Max(1000)`." ([PR #4239](https://github.com/opensearch-project/data-prepper/pull/4239#discussion_r1513597264))
- **Cross-field constraints (mutual exclusivity, conditional requirements)
  are `@AssertTrue` methods with a message on the config class, not imperative
  checks in the consuming class.**
  Evidence: "Put validations in the `Config` classes themselves. You can use an `@AssertTrue` from the Jakarta package." ([PR #6659](https://github.com/opensearch-project/data-prepper/pull/6659#discussion_r3009804041))
- **Defaults live in the config class itself (field initializer / non-null
  getter), never duplicated at call sites; whatever a config check can't
  express declaratively runs in the constructor so the pipeline fails fast at
  startup, not on the first event.**
  Evidence: "We should verify that the codec is one we support in the constructor. This allows the processor validation to fail fast." ([PR #5032](https://github.com/opensearch-project/data-prepper/pull/5032#discussion_r1813792978)), "It would be ideal to have the config return a non-null value. Then you won't need a bunch of null checks cluttering the code." ([PR #6146](https://github.com/opensearch-project/data-prepper/pull/6146#discussion_r2414343843))

## Logging discipline

- **SLF4J `{}` interpolation only — no string concatenation, no
  `String.format` inside log calls, and the format string must be a constant
  (a dynamic string passed as the first argument is treated as a format
  string). No `System.out.println` or `e.printStackTrace()` anywhere,
  including tests.**
  Evidence: "You should not use a `String.format` inside a log statement." ([PR #3075](https://github.com/opensearch-project/data-prepper/pull/3075#discussion_r1280644199)), "Unknown strings shouldn't be passed to a logger since it is treated as a format string." ([PR #6184](https://github.com/opensearch-project/data-prepper/pull/6184#discussion_r2452281183))
- **Distinguish expected from unexpected exceptions: expected errors (bad
  user input, throttling, access denied) are logged message-only without a
  stack trace; unexpected errors get ERROR level with the exception object as
  the last argument so the stack trace is included. Never log both `e` and
  `e.getMessage()` (redundant).**
  Evidence: "Find expected exceptions and handle differently from unexpected exceptions. Then use the stack trace for the unexpected only." ([PR #5713](https://github.com/opensearch-project/data-prepper/pull/5713#discussion_r2103297689)), "You don't need `e.getMessage()` as it will be included by just `e`." ([PR #6273](https://github.com/opensearch-project/data-prepper/pull/6273#discussion_r2586804425))
- **Per-record/per-iteration logs in hot paths are DEBUG or TRACE, never
  INFO; messages carry identifying context (id, shard, pipeline name) and a
  stable searchable string; no trailing ellipses ("...") and no exclamation
  marks.**
  Evidence: "Let's make this a `debug` level log. Otherwise, we'll have too many logs generated." ([PR #4831](https://github.com/opensearch-project/data-prepper/pull/4831#discussion_r1722110448)), "Let's avoid these ellipses at the end. They are unclear and make it seem that more is coming." ([PR #4603](https://github.com/opensearch-project/data-prepper/pull/4603#discussion_r1629724662))
- **Never log secrets or credential values — log a non-sensitive identifier
  instead; prefer a metric over a log line for measurable behavior.**
  [repo-specific: high-volume or sensitive logs use the `NOISY`/`SENSITIVE`
  markers]
  Evidence: "We should not be logging the secret. Maybe log the secret ARN." ([PR #6105](https://github.com/opensearch-project/data-prepper/pull/6105#discussion_r2401948839)), "Should we report this as a metric instead of log it?" ([PR #3614](https://github.com/opensearch-project/data-prepper/pull/3614#discussion_r1390294395))

## Error handling is proportional and informative

- **A recoverable error in one component must never terminate the whole
  process or let a worker loop die (`System.exit` is out; log, back off,
  retry); conversely, startup/config failures that make the component
  unusable must rethrow so it fails fast rather than limping into the data
  path.**
  Evidence: "This will exit all of Data Prepper. That is a far too severe consequence of an error communicating with SQS." ([PR #1431](https://github.com/opensearch-project/data-prepper/pull/1431#discussion_r883082012)), "we should re-throw because this will cause the pipeline to fail to run. As it is now, the pipeline would start, all processing would fail" ([PR #4750](https://github.com/opensearch-project/data-prepper/pull/4750#discussion_r1682923494))
- **One bad item must not fail its whole batch — exception handling goes
  inside the per-item loop; and no-op remote calls (e.g. an empty delete
  batch) must be skipped, not sent.**
  Evidence: "Say we get a metrics export request with 50 metrics and only one is invalid. The current try-catch will fail them all." ([PR #2176](https://github.com/opensearch-project/data-prepper/pull/2176#discussion_r1086010613))
- **Catch and throw specific types: `catch (SpecificException e)` not
  `instanceof` branching; wrap checked exceptions in a domain exception; user
  configuration errors throw the project's config-exception type. Error
  messages are complete sentences that include the offending value and name
  the relevant option(s).**
  Evidence: "My comment was more on the use of `if(e instanceof SpecificException)`. I'd rather: `catch(SpecificException e)`." ([PR #805](https://github.com/opensearch-project/data-prepper/pull/805#discussion_r784082330)), "throw new IllegalArgumentException(String.format(\"The value provided for sts_role_arn is not a valid AWS ARN. Provided value: %s\", awsStsRoleArn));" ([PR #5274](https://github.com/opensearch-project/data-prepper/pull/5274#discussion_r1900176131))
- **When input data is unsupported, fail fast with an exception naming the
  offending type/value — never silently produce wrong output (e.g.
  `toString()` of a List).**
  Evidence: "when something is not supported we should fail-fast." ([PR #2528](https://github.com/opensearch-project/data-prepper/pull/2528#discussion_r1175684952))

## Minimal visibility by default

- **Start closed, open only as needed: classes are package-private unless a
  caller outside the package exists; fields are private (constants
  `private static final`) with getters; members made public only so tests can
  reach them become package-private instead; methods for subclasses are
  `protected`, not public; single-consumer helper classes become
  `private static` nested classes.**
  Evidence: "Can these classes be package protected? If so, let's start there and open only as needed." ([PR #6539](https://github.com/opensearch-project/data-prepper/pull/6539#discussion_r2841747896)), "Do we need this? If it is only for testing, please make it package protected (remove the `public` modifier)." ([PR #2198](https://github.com/opensearch-project/data-prepper/pull/2198#discussion_r1090778244))
- **Do not promote a local variable to a mutable instance field for
  convenience (thread-safety hazard); per-call state stays local.**
  Evidence: "This is not thread-safe. Is there a particular reason you moved it to the field? I say we just keep this as a local variable." ([PR #4327](https://github.com/opensearch-project/data-prepper/pull/4260#discussion_r1524061552))

## Design: small testable classes, strategy over conditionals

- **Multi-concern logic embedded in a worker/processor method (parsing,
  retries, conversion, signing) moves into its own small class with its own
  unit tests; the host class mocks it.**
  Evidence: "We can simplify this by moving the header extraction into its own class. Then you can cover all the complexities in the unit tests for that class." ([PR #6507](https://github.com/opensearch-project/data-prepper/pull/6507#discussion_r2814208953)), "This is very important logic and is currently not thoroughly tested. This could exist in a different class. Then it would highly testable." ([PR #2623](https://github.com/opensearch-project/data-prepper/pull/2623#discussion_r1186177634))
- **Replace if/else or switch dispatch on a type/format/mode with a strategy
  interface and one implementation per variant; prefer interfaces and
  composition over abstract base classes; do not add mode booleans that
  branch inside shared code — add a new concrete implementation instead.**
  Evidence: "We should use the strategy pattern to inject code for handling the parsed events based on aggregate or strict. Then you don't need this large conditional." ([PR #5032](https://github.com/opensearch-project/data-prepper/pull/5032#discussion_r1813790542)), "Rather than an abstract class, make this an interface. It is more flexible and allows for adapting rather than having to convert." ([PR #5095](https://github.com/opensearch-project/data-prepper/pull/5095#discussion_r1815500898))
- **Collaborators (SDK clients, converters) are injected through the
  constructor, never constructed inside methods — for testability and final
  fields. Declare the narrowest type that works (`OutputStream` not
  `BufferedOutputStream`, `Collection` not `List` when order is irrelevant,
  `Supplier<T>` to decouple).**
  Evidence: "I think we should pass this in as a parameter. This will let you more easily mock this for unit tests." ([PR #2320](https://github.com/opensearch-project/data-prepper/pull/2320#discussion_r1120426200)), "I'd recommend making this just an `OutputStream` so that the code doesn't have to change if you change this stream class." ([PR #2645](https://github.com/opensearch-project/data-prepper/pull/2645#discussion_r1194231255))
- **Never use double-brace initialization (`new X() {{ ... }}`); nontrivial
  anonymous classes become named package-private classes with their own
  tests. Public methods do not return `Pair` (make a small named model) and
  never take or store `Optional` (parameter/field anti-pattern — take the
  nullable type and wrap only on return).**
  Evidence: "This (`{{`) is double brace initialization and is an anti-pattern." ([PR #2034](https://github.com/opensearch-project/data-prepper/pull/2034#discussion_r1029895468)), "Passing an `Optional` as an argument is an anti-pattern. Java does nothing to prevent us from passing in `null`." ([PR #3916](https://github.com/opensearch-project/data-prepper/pull/3916#discussion_r1442991624)), "Let's avoid `Pair` classes on public methods because `left` and `right` don't have much meaning outside the pair." ([PR #6229](https://github.com/opensearch-project/data-prepper/pull/6229#discussion_r2511547735))
- **No grab-bag `Constants`/`Util` classes: constants live in the class that
  uses them; shared behavior is an injected instance (DI), not static
  methods.**
  Evidence: "Rather than a big `Constants` class move constants into the classes that need them." ([PR #5125](https://github.com/opensearch-project/data-prepper/pull/5125#discussion_r1825178770)), "Let's not use `Util` classes. We support Spring dependency injection. So use that instead." ([PR #5767](https://github.com/opensearch-project/data-prepper/pull/5767#discussion_r2103274773))
- **[repo-specific] Code reads events/metrics through the
  `data-prepper-api` interfaces (`Event`, `Gauge`, …), never the `JacksonX`
  implementation classes except at construction; new sources create events
  via an injected `EventFactory`. Each plugin's classes live in a unique Java
  package (identical packages across plugin jars clobber classes at runtime),
  and plugins must not depend on `data-prepper-core`.**
  Evidence: "Do not rely on the `JacksonX` implementations. Use the interfaces directly." ([PR #6229](https://github.com/opensearch-project/data-prepper/pull/6229#discussion_r2495270339)), "Please move all of these files into a unique package. This can lead to conflicts which break functionality at runtime." ([PR #2995](https://github.com/opensearch-project/data-prepper/pull/2995#discussion_r1277774361))

## Reuse before reinventing; consolidate duplication

- **Code copied or near-duplicated from another class/plugin must be
  consolidated (shared method, class, or common module) rather than merged as
  a parallel copy — divergence causes missed fixes. Large consolidations may
  be deferred to a follow-on PR only with a tracking issue.**
  Evidence: "Having duplicate classes is going to be very difficult to maintain as we fix bugs. We should consolidate this into a library." ([PR #5274](https://github.com/opensearch-project/data-prepper/pull/5274#discussion_r1900180369)), "We should not fork this. The primary difference is that we need to sample the document." ([PR #6990](https://github.com/opensearch-project/data-prepper/pull/6990#discussion_r3856639871))
- **Before writing new retry/backoff, HTTP-client, thread-factory,
  reflection, or credential code, use the existing project/library component.**
  [repo-specific examples: Armeria `Backoff`, `BackgroundThreadFactory`,
  `ReflectivelySetField`, `TestEventFactory`, `AwsCredentialsSupplier`]
  Evidence: "Armeria has a utility that we can use for backoffs and retries. Can you make use of that?" ([PR #5664](https://github.com/opensearch-project/data-prepper/pull/5664#discussion_r2070135335)), "Let's use the `BackgroundThreadFactory` for all `ExecutorServices` going forward." ([PR #4379](https://github.com/opensearch-project/data-prepper/pull/4379#discussion_r1546888886))
- **Structured output is built with the proper builder — JSON via Jackson
  (never `StringBuilder` concatenation), URLs via a URI builder (never string
  manipulation).**
  Evidence: "We should build the JSON using Jackson instead of `StringBuilder` to avoid errors." ([PR #6691](https://github.com/opensearch-project/data-prepper/pull/6691#discussion_r3053436927))

## Dead code and PR hygiene

- **Commented-out code, unused fields/methods/dependencies, leftover debug
  statements, and AI-process artifacts (bootstrap histories, checkbox/task
  notes, "AI notes" about how the code was generated) are deleted — Git
  history preserves them. Judge comments by their content, not their
  provenance: a comment that genuinely aids the next reader is acceptable
  regardless of whether a human or an AI wrote it; what gets removed is
  process residue and anything the surrounding code-comment principles below
  would reject anyway.**
  Evidence: "Please remove all of these comments. We don't keep comments like this in the code base. We have a full Git history." ([PR #2289](https://github.com/opensearch-project/data-prepper/pull/2289#discussion_r1112285556)), "These look like random AI generated notes. Do we really need this here?" ([PR #6479](https://github.com/opensearch-project/data-prepper/pull/6479#discussion_r2774512165))
- **PRs contain no unrelated diffs: no commits from other PRs, no
  whitespace-only changes, no already-merged changes from a missed rebase.**
  Evidence: "This appears unrelated and appears to be from another PR. Please remove this change." ([PR #4818](https://github.com/opensearch-project/data-prepper/pull/4818#discussion_r1714022174))
- **A comment that explains a block of code becomes an extracted well-named
  method or a Javadoc; comments annotating a member use Javadoc (`/** */`) so
  IDEs keep them attached; section-divider comments in test classes become
  `@Nested` classes or `@DisplayName` or are removed; never restate a
  constant's value in nearby prose or docs where it can drift.**
  Evidence: "Rather than add an inline comment (which is prone to being lost over time), I suggest you make this a function." ([PR #1224](https://github.com/opensearch-project/data-prepper/pull/1224#discussion_r836751152)), "If we change the constant value to 5_000_000L to 6_000_000L we might miss that string \"5MB\"" ([PR #2179](https://github.com/opensearch-project/data-prepper/pull/2179#discussion_r1091261236))
- **No binary files checked into the repo (keystores, compressed fixtures,
  jars): generate test fixtures during setup or pull via the build.**
  Evidence: "Let's not commit JKS files. We should be able to generate them for test setup." ([PR #6755](https://github.com/opensearch-project/data-prepper/pull/6755#discussion_r3173278790))

## Magic numbers and literals

- **Unexplained numeric/string literals become `static final` constants whose
  names state their meaning; derive the value from its source where possible
  (`"stream/".length()` instead of `7`); a comment next to a literal is not an
  acceptable substitute; large literals use underscores (`1_000_000`).**
  Evidence: "This `4` is a magical number. Please make it a constant with an understandable name." ([PR #4285](https://github.com/opensearch-project/data-prepper/pull/4285#discussion_r1525530617)), "Please `HTTP_METHODS.size()` instead of `4`. These could easily change and the constant might be missed." ([PR #1548](https://github.com/opensearch-project/data-prepper/pull/1548#discussion_r909973354))
- **An expression used two or more times is extracted into a named variable;
  chained calls like `list.get(0).getX()` in assertions are split with
  intermediate assertions so an NPE source is traceable.**
  Evidence: "Please extract a variable for this ... You use it below." ([PR #559](https://github.com/opensearch-project/data-prepper/pull/559#discussion_r746126591))

## Hot-path efficiency

- **Objects that don't vary per call — `Pattern.compile`, `ObjectMapper`,
  `DateTimeFormatter`, SDK clients, codecs — are created once in the
  constructor or as `static final` fields, never per invocation or inside
  loops; avoid `String.format` in hot paths; pre-size collections when the
  size is known; batch writes instead of per-item calls.**
  Evidence: "This will compile `\"\\\\s\"` each time. Instead compile the regex in the constructor." ([PR #3349](https://github.com/opensearch-project/data-prepper/pull/3349#discussion_r1342894040)), "`String.format` is quite slow. As this will be run repeatedly, let's use string addition." ([PR #2696](https://github.com/opensearch-project/data-prepper/pull/2696#discussion_r1204439613))
- **Caches/maps populated from unbounded input must be bounded (e.g. LRU with
  a configurable max).**
  Evidence: "As it is now, this `Map` can grow unbounded. ... Make this into an LRU cache" ([PR #1335](https://github.com/opensearch-project/data-prepper/pull/1335#discussion_r861999293))

## Concurrency

- **No mutable static state in component classes — statics are shared across
  every instance in the JVM (all pipelines); fields written by one thread and
  read by another are `volatile` or otherwise safely published.**
  Evidence: "Since this is static, it will stop all consumers in all pipelines. Use an instance per pipeline instead." ([PR #3349](https://github.com/opensearch-project/data-prepper/pull/3349#discussion_r1342905210)), "Since you are changing this on a different thread, you should make the field `volatile`." ([PR #2324](https://github.com/opensearch-project/data-prepper/pull/2324#discussion_r1123974988))
- **No `while(true)`/unbounded retry loops deep in the call stack — the loop
  belongs in the thread's top-level `Runnable`, with shutdown via a boolean
  flag set by a `shutdown()` method (not thread interruption). Threads are
  named for debuggability and non-core threads are daemon threads.**
  Evidence: "Placing loops that run like this deep in the call stack can make debugging very difficult. We should move the loop into the `Runnable` method." ([PR #4277](https://github.com/opensearch-project/data-prepper/pull/4277#discussion_r1524028280)), "1. Name the threads to help with debugging. 2. Make the threads daemon threads since they are not doing core processing." ([PR #2521](https://github.com/opensearch-project/data-prepper/pull/2521#discussion_r1170674128))
- **AutoCloseable resources use try-with-resources, not manual `close()`
  calls an exception can skip.**
  Evidence: "This could be missed if you have an exception that isn't caught. I recommend you use try-with-resources." ([PR #4105](https://github.com/opensearch-project/data-prepper/pull/4105#discussion_r1487091317))

## Backward compatibility of released behavior

- **[repo-specific] A change must not alter defaults, output shape, tagging,
  or metric names that a released version already exposes: keep the old
  default, alias a renamed option, gate new behavior behind explicit opt-in,
  or label the PR a breaking change for the next major version.**
  Evidence: "This configuration was already released in Data Prepper 2.9, so this is a breaking change." ([PR #5041](https://github.com/opensearch-project/data-prepper/pull/5041#discussion_r1796136268)), "This is a breaking change since we already have this metric." ([PR #6488](https://github.com/opensearch-project/data-prepper/pull/6488)), "We should keep the same defaults so that an in-place version update doesn't have a performance impact." ([PR #7194](https://github.com/opensearch-project/data-prepper/pull/7194#discussion_r4303943939))
- **[repo-specific] New plugins whose configuration surface may still change
  carry the `@Experimental` annotation; new public API Javadocs carry
  `@since <next-release-version>`.**
  Evidence: "Let's add the `@Experimental` annotation to this class since this is new and still under development." ([PR #5406](https://github.com/opensearch-project/data-prepper/pull/5406#discussion_r1943466553)), "Please add `@since 1.3` to the Javadocs since this is a public interface." ([PR #825](https://github.com/opensearch-project/data-prepper/pull/825#discussion_r779028697))

## Gradle build hygiene

- **Sub-project build files never redeclare what the root provides: no
  dependency version numbers covered by a BOM/catalog, no repeated
  `plugins`/`repositories`/test blocks, no `group`/`version`; never lower an
  existing dependency version; logging implementations live only in the core
  project (subprojects use the SLF4J API).**
  Evidence: "Please do not specify the versions for the AWS SDK. The root project includes the AWS SDK BOM." ([PR #5664](https://github.com/opensearch-project/data-prepper/pull/5664#discussion_r2100525640)), "You don't need this block. It is inherited." ([PR #4181](https://github.com/opensearch-project/data-prepper/pull/4181#discussion_r1500970607))
- **Test-only dependencies are `testImplementation`; every declared
  dependency must be used — remove speculative ones.**
  Evidence: "Please make these three dependencies available only as `testImplementation`." ([PR #2309](https://github.com/opensearch-project/data-prepper/pull/2309#discussion_r1147774089)), "I see a lot of dependencies here which do not seem to be needed." ([PR #2661](https://github.com/opensearch-project/data-prepper/pull/2661#discussion_r1188590649))

## Metrics conventions

- **Metric names state exactly what is measured, in lowerCamelCase, without
  the plugin/scope name (auto-prefixed) and without a type suffix like
  `Counter`; a new metric must not duplicate an existing metric's concept —
  reuse it.**
  Evidence: "Why did you add `decompression` in this name? It will already be scoped to the processor" ([PR #4118](https://github.com/opensearch-project/data-prepper/pull/4118#discussion_r1487082970)), "Perhaps `recordsWriteFailed` would be clearer? We are not overflowing the buffer, but instead failed to write to the buffer." ([PR #2170](https://github.com/opensearch-project/data-prepper/pull/2170#discussion_r1084344752))
- **Use the semantically correct instrument: `DistributionSummary` for sizes
  and distributions, counter for monotonic counts (never a gauge just to
  decrement), gauges registered once against a tracked object. Failure
  counters increment inside the branch that handles the failure, not in
  separate classification logic that can drift.**
  Evidence: "I think request size needs to be a distribution summary." ([PR #6183](https://github.com/opensearch-project/data-prepper/pull/6183#discussion_r2440558147)), "Let's just put the `.increment()` calls in the conditions where you handle the exceptions." ([PR #6238](https://github.com/opensearch-project/data-prepper/pull/6238#discussion_r2495026047))

## Naming refinements (beyond general "descriptive names")

- **Method names match their side effects: `get*`/`is*` must not mutate state
  or make remote calls; boolean-returning methods read as predicates
  (`isX`/`hasX`/`areXEnabled`), not imperatives.**
  Evidence: "A better name might be `popEvent`. Normally, a `get` method does not mutate the object." ([PR #2982](https://github.com/opensearch-project/data-prepper/pull/2982#discussion_r1258490296)), "Java conventions are normally to make a boolean method more like a question. So this could be `areAcknowledgementsEnabled`." ([PR #2436](https://github.com/opensearch-project/data-prepper/pull/2436#discussion_r1158525904))
- **No `I`-prefix on interfaces (name by capability) and no `Impl` suffix on
  implementations (name by distinguishing strategy/technology, e.g.
  `JacksonEvent`, `DefaultX`); acronyms in camelCase are capitalized as words
  (`Aws` not `AWS`); enum constants are ALL_CAPS with the serialized string
  as a constructor field; multi-word package segments use underscores.**
  Evidence: "The `IInterfaceName` convention is a .NET convention and not used often in Java." ([PR #4118](https://github.com/opensearch-project/data-prepper/pull/4118#discussion_r1487083268)), "I'd like to drop the `Impl` from this name and make it a `JacksonEvent`." ([PR #435](https://github.com/opensearch-project/data-prepper/pull/435#discussion_r730007119))

## Security checks on inputs and CI

- **User-supplied values used in paths must reject `..` and verify the
  resolved path stays under the base directory; values interpolated into
  SQL/CQL or follow-up URLs must be escaped or validated; externally-supplied
  pagination links are scrubbed and bounded (a malicious or buggy server must
  not cause an infinite loop).**
  Evidence: "Calling `resolve` can allow for `..` which could allow invalid input to resolve to files in other directories." ([PR #6682](https://github.com/opensearch-project/data-prepper/pull/6682#discussion_r3058783307)), "To prevent SQL injection, we need regex validation on the table name." ([PR #6757](https://github.com/opensearch-project/data-prepper/pull/6757#discussion_r3190095217))
- **Do not weaken security defaults even in demos/examples (TLS hostname
  validation stays on; https required unless an explicit insecure flag is
  set); no real resource names, usernames, or credentials in code or
  templates.**
  Evidence: "we intentionally use TLS hostname validation in this demo. We should continue to use it here." ([PR #4972](https://github.com/opensearch-project/data-prepper/pull/4972#discussion_r1843857979))
- **CI workflows must not run unverified PR code with access to secrets;
  third-party GitHub Actions are pinned by commit SHA.**
  Evidence: "This is not secure because it runs the unverified code from the PR in this GitHub Action which has access to GitHub Secrets." ([PR #6866](https://github.com/opensearch-project/data-prepper/pull/6866#discussion_r3267489203)), "We've recently moved to using SHAs instead of friendly version numbers" ([PR #6939](https://github.com/opensearch-project/data-prepper/pull/6939#discussion_r3493938218))

## Null/Optional refinements (beyond "Optional for return values")

- **Never `Optional` as a parameter or field; never `.get()` without a
  presence check (prefer `orElseThrow`/`ifPresent`/`map`); don't wrap a value
  in Optional merely to call `ifPresent` where a plain null check is clearer;
  outside stream/`Predicate` contexts use `x == null`, not
  `Objects.isNull/nonNull`.**
  Evidence: "Taking an `Optional` as an argument is not a good practice. You have no non-null guarantees here, so you should do a null check, which somewhat defeats the purpose." ([PR #846](https://github.com/opensearch-project/data-prepper/pull/846#discussion_r781399851)), "The reason `isNull` exists is to help with streams. I think outside of a stream, we should use `x == null`." ([PR #3532](https://github.com/opensearch-project/data-prepper/pull/3532#discussion_r1376576249))

## Documentation prose

- **Proper nouns are correctly capitalized ("Docker", "Apache", the project's
  own name); hyperlink text is the target's title, never the word "here".**
  Evidence: "Please capitalize \"docker\" to \"Docker\"." ([PR #729](https://github.com/opensearch-project/data-prepper/pull/729#discussion_r817231237)), "I recommend using the title of the article rather than \"here\" on links." ([PR #704](https://github.com/opensearch-project/data-prepper/pull/704#discussion_r763183385))

---

## Lower-confidence principles (2 PRs of evidence — operator judgment)

- **Javadoc asterisks align in one column, one `*` per line.**
  Evidence: ([PR #2324](https://github.com/opensearch-project/data-prepper/pull/2324#discussion_r1123972003)), ([PR #1620](https://github.com/opensearch-project/data-prepper/pull/1620#discussion_r930229066))
- **[repo-specific] `@JsonProperty` on config fields with defaults declares
  `defaultValue` (as a string) for documentation generation; example-value
  annotations are omitted on enum-typed options (the enum list is the full
  set of examples).**
  Evidence: ([PR #4294](https://github.com/opensearch-project/data-prepper/pull/4294#discussion_r1823036547)), ([PR #5133](https://github.com/opensearch-project/data-prepper/pull/5133#discussion_r1824718963))

## Process/tone observations (not diff-checkable; for prompt calibration)

- Larger refactors are routinely deferred: "fine as a follow-on PR, but
  please create a GitHub issue to track it." Non-blocking feedback is labeled
  (`nit:`, "I won't block the PR for this"); firm asks say "Please ...";
  uncertain ones are questions.
- Blocking-firm topics in practice: security, backward compatibility,
  vacuous/missing tests, and infinite-loop/process-exit error handling.
- He supplies ready-to-paste replacement code or a link to an in-repo
  exemplar file with most asks.
- Bot rule: do not thank, praise, or congratulate contributors — that is the
  human reviewer's voice, not the bot's. Keep comments to the technical ask.
