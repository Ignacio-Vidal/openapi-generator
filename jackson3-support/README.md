# Jackson 3 support for `java/microprofile` and `jaxrs-spec` (`quarkus`)

> **Review-only file.** This README carries the plan and the draft upstream issue/PR text for review
> in the fork. It must be dropped from the branch before the PR is opened upstream.

## 1. Why: Quarkus 4 moves to Jackson 3

[Quarkus 4.0.0.Beta1](https://quarkus.io/blog/quarkus-4-0-0-beta1-released/) (final planned for end of
November 2026) moves from Jackson 2 to Jackson 3 ([migration guide](https://github.com/quarkusio/quarkus/wiki/Migration-Guide-4.0)):

- `com.fasterxml.jackson.core` → `tools.jackson.core`, `com.fasterxml.jackson.databind` → `tools.jackson.databind`
  (incl. `databind.annotation.JsonDeserialize`), and `jackson-dataformat-xml` → `tools.jackson.dataformat.xml`
  (its annotations too).
- **Annotations are unchanged**: `com.fasterxml.jackson.annotation.*` and the `jackson-annotations` artifact stay
  on their own 2.x line (2.21).
- `jackson-datatype-jsr310` is gone; java.time support is built into `jackson-databind` 3
  (`tools.jackson.databind.ext.javatime`).
- Quarkus produces a `JsonMapper` bean; `ObjectMapperCustomizer` → `JsonMapperBuilderCustomizer`.
- Jackson 3 needs Java 17; Quarkus 4 needs Java 21.

Quarkus 4.0.0.Beta1 resolves `tools.jackson.core:jackson-databind:3.1.4` and
`com.fasterxml.jackson.core:jackson-annotations:2.21`.

### What broke when upgrading a real project

Upgrading [openapi-servers](https://github.com/Ignacio-Vidal/openapi-servers) (4 `jaxrs-spec`/`quarkus` server
tasks + 1 `java`/`microprofile` client task, all oneOf/sealed/records/bean-validation options on) with
`quarkus update` to 4.0.0.Beta1:

| Generated code | Result on Quarkus 4 |
|---|---|
| `jaxrs-spec` / `quarkus` models + APIs | **Compiles** — this spec only produces `com.fasterxml.jackson.annotation.*` imports |
| `java` / `microprofile` `RFC3339DateFormat`, `RFC3339InstantDeserializer`, `RFC3339JavaTimeModule` | **25 compile errors** — `SimpleModule`, `StdDateFormat`, `jsr310.InstantDeserializer`, `JsonParser`, `DeserializationContext` no longer exist under `com.fasterxml.jackson` |

Workaround used in the project until the generator supports it — skip the (unreferenced) helpers:

```kotlin
openapiGeneratorIgnoreList.set(listOf("src/main/java/com/example/openapi/quarkus/client/RFC3339*.java"))
```

The server side only survived because this spec has no `uniqueItems` arrays: those make `jaxrs-spec` emit
`import com.fasterxml.jackson.databind.annotation.JsonDeserialize;`, which does not compile on Jackson 3.

## 2. Current state upstream (master `5579e3a322a`)

| Generator / library | `useJackson3` | Notes |
|---|---|---|
| `java` — native, apache-httpclient, jersey3, rest-assured, restclient\*, resttemplate\*, webclient\* | ✅ | \* Spring libs require `useSpringBoot4=true`. Shared `Java/RFC3339*.mustache` already branch on `{{#useJackson3}}` / `{{jacksonPackage}}` |
| `java` — **microprofile** | ❌ rejected | `IllegalArgumentException: useJackson3 is only supported for ...` → **this PR** |
| `jaxrs-spec` (default library) | 🟡 in flight | Issue [#25363](https://github.com/OpenAPITools/openapi-generator/issues/25363), PR [#25406](https://github.com/OpenAPITools/openapi-generator/pull/25406) (open, by another contributor) |
| `jaxrs-spec` — **quarkus** library | ❌ | #25406 explicitly **rejects** `useJackson3` for every non-default library (`testUseJackson3RejectsNonDefaultLibrary` uses `quarkus`) |
| `java` — resteasy | ❌ | Issue [#25364](https://github.com/OpenAPITools/openapi-generator/issues/25364) |

## 3. Plan

### A. `java` / `microprofile` — implemented in this branch

| Area | Change |
|---|---|
| `JavaClientCodegen` | Allow `useJackson3` for `microprofile`. Require `microprofileRestClientVersion=3.0` (the 1.4.1/2.0 pom targets javax + Java 8; Jackson 3 needs 17) and `serializationLibrary=jackson` (JSON-B is meaningless with it) — both fail fast with an `IllegalArgumentException`. Force `java17`. Keep generating the RFC3339 helpers (the shared templates already compile on Jackson 3). Option + library descriptions updated. |
| `microprofile/pom_3.0.mustache` | `jackson-core`/`jackson-databind` groupId → `{{jacksonPackage}}.core`; `jackson-dataformat-xml` → `{{jacksonPackage}}.dataformat`; `jackson-annotations` stays `com.fasterxml.jackson.core` on its own `jackson-annotations.version` 2.21; `jackson.version` 3.1.6 (same as native/apache/jersey3/rest-assured); drop `jackson-datatype-jsr310` under Jackson 3. |
| `microprofile/model.mustache` | `withXml` import → `{{jacksonPackage}}.dataformat.xml.annotation.*` |
| imports | `JsonDeserialize` (emitted for `uniqueItems` setters) already maps to `tools.jackson.databind.annotation` via the existing `useJackson3` import mapping — no change needed |
| Tests (`JavaClientCodegenTest`) | `testMicroprofileWithJackson3` (asserts **no** generated `.java` file references `com.fasterxml.jackson.{core,databind,datatype}`, RFC3339 helpers on `tools.jackson`, pom), `…UsesJackson3JsonDeserializeForSets`, `…AndXml`, `testMicroprofileWithJackson2IsUnchanged`, `…RejectsJavaxRestClient`, `…RejectsJsonb`. Mutation-checked: reverting the XML import, the jsr310 guard, or the RFC3339 condition each fails exactly one test. |
| Sample + CI | `bin/configs/java-microprofile-rest-client-3.0-jackson3.yaml` → `samples/client/petstore/java/microprofile-rest-client-3.0-jackson3`, built in `samples-java-client-jdk17.yaml` |
| Docs | `docs/generators/java.md`, `java-microprofile.md` regenerated |

Verification done:
- New sample `mvn test` on JDK 21 against its own pom; dependency tree is `tools.jackson.core:jackson-{core,databind}:3.1.6` + `jackson-annotations:2.21`, no Jackson 2 databind.
- All 14 existing `java-microprofile*` samples regenerate **byte-identical** (Jackson 2 default unchanged).
- End to end: openapi-servers on Quarkus 4.0.0.Beta1 with the client regenerated by this branch
  (`useJackson3=true`, no ignore-list workaround, RFC3339 helpers compiled) — **58/58 tests green**
  (PetApiTest 24, SecurityApiTest 28 with Keycloak Dev Services, FileUploadApiTest 6).

### B. `jaxrs-spec` (default library) — do NOT duplicate

Already covered by #25363 / #25406 (adds `useJackson3`, `jacksonPackage`, `JsonDeserialize` mapping,
`Shape.POJO` vs `OBJECT`, Jackson 3 provider groupIds, Java 17). Action: review/support #25406 rather than
open a competing PR. Checked: both `tools.jackson.jaxrs:jackson-jaxrs-json-provider` and
`tools.jackson.jakarta.rs:jackson-jakarta-rs-json-provider` exist on Central, so its pom is resolvable.

### C. `jaxrs-spec` / `quarkus` — follow-up, stacked on #25406 after it merges

1. Lift the non-default-library rejection for `quarkus` only (keep rejecting thorntail/kumuluzee/openliberty/helidon,
   whose poms are already broken for `useJakartaEe`).
2. Require `useJakartaEe=true` with `useJackson3` for `quarkus` (javax path is Quarkus 1.13).
3. `libraries/quarkus/pom.mustache`: with `useJackson3`, emit the Quarkus 4 platform (`quarkus.platform.version` /
   `quarkus-plugin.version` → 4.0.x once final; Beta1 today) and `maven.compiler.release` 21. Jackson versions
   come from the Quarkus BOM — no explicit Jackson deps.
4. Sources need nothing beyond #25406 (`JsonDeserialize` import mapping, `Shape.POJO`).
5. **Verify at runtime** before merging: `openApiNullable=true` — `jackson-databind-nullable` 0.2.13 ships
   `JsonNullableJackson3Module` registered via `META-INF/services/tools.jackson.databind.JacksonModule`; confirm
   Quarkus 4 discovers it (Kotlin hit the inverse problem: #24859).
6. New sample `jaxrs-spec-quarkus-jackson3` in `samples-jdk17.yaml` (needs a JDK 21 job — Quarkus 4 minimum).

### D. Follow-ups / out of scope

- [#24291](https://github.com/OpenAPITools/openapi-generator/pull/24291) (microprofile Gradle build, open): whichever
  merges second must apply the same Jackson 2/3 switch to `microprofile/build.gradle.mustache`.
- `USE_JACKSON_3` / `JACKSON*_PACKAGE` constants are duplicated across `JavaClientCodegen`, `SpringCodegen`,
  `JavaJAXRSSpecServerCodegen` (#25406), Kotlin — consolidate into `AbstractJavaCodegen` (also suggested in #25406).
- Microprofile javax (`pom.mustache`, MP Rest Client 1.4.1/2.0) intentionally not supported with Jackson 3.

---

## 4. Draft upstream issue

**Title:** `[REQ][Java][microprofile] Support useJackson3 (Quarkus 4 / Jackson 3)`

```markdown
### Is your feature request related to a problem? Please describe.

Quarkus 4 (4.0.0.Beta1 out, final planned for end of November) moves to Jackson 3: `jackson-core`/`jackson-databind`
are now `tools.jackson.*`, `jackson-datatype-jsr310` is folded into databind, while `com.fasterxml.jackson.annotation`
is unchanged.

A `java` client generated with `library=microprofile`, `serializationLibrary=jackson` no longer compiles on Quarkus 4:
the `RFC3339DateFormat`, `RFC3339InstantDeserializer` and `RFC3339JavaTimeModule` supporting files import
`com.fasterxml.jackson.databind.module.SimpleModule`, `...databind.util.StdDateFormat`,
`com.fasterxml.jackson.datatype.jsr310.deser.InstantDeserializer`, ... (25 compile errors in my project).
`withXml` models and `uniqueItems` setters (`@JsonDeserialize`) break the same way.

`useJackson3` already exists for the `native`, `apache-httpclient`, `jersey3`, `rest-assured` and Spring
libraries, but is rejected for `microprofile`:

    useJackson3 is only supported for the 'native', 'apache-httpclient', 'jersey3', 'rest-assured', 'restclient', 'resttemplate', and 'webclient' libraries.

### Describe the solution you'd like

Support `useJackson3=true` for `library=microprofile`, Jackson 2 staying the default:

- allow the option for microprofile with `microprofileRestClientVersion=3.0` (jakarta; Jackson 3 needs Java 17) and
  `serializationLibrary=jackson`, failing fast otherwise
- `pom_3.0.mustache`: `tools.jackson.core` / `tools.jackson.dataformat` groupIds, `jackson-annotations` on its 2.x line,
  no `jackson-datatype-jsr310`, Java 17
- `model.mustache`: Jackson 3 XML annotation package
- the shared `RFC3339*` templates and the `JsonDeserialize` import mapping already handle `useJackson3`

### Describe alternatives you've considered

Excluding the RFC3339 helpers via `.openapi-generator-ignore` (nothing references them in a Quarkus app, which
configures its own `JsonMapper`) works for my spec, but not for `withXml` or `uniqueItems` models.

### Additional context

- `jaxrs-spec` counterpart: #25363 / #25406. Note #25406 rejects `useJackson3` for `library=quarkus`; I'm happy to
  follow up with Quarkus 4 support for that library once it is merged.
- Related: #22294 (Jackson 3 for Java libraries), #25364 (resteasy).

I have a PR ready.
```

## 5. Draft upstream PR

**Title:** `[Java][microprofile] support useJackson3`

```markdown
Fixes #<issue>

Adds Jackson 3 support to the `java` generator's `microprofile` library via the existing `useJackson3` option, so
generated clients compile on Quarkus 4 (Jackson 3). Jackson 2 stays the default and its output is unchanged
(all `java-microprofile*` samples regenerate identically).

- `JavaClientCodegen`: allow `useJackson3` for microprofile; requires `microprofileRestClientVersion=3.0` and
  `serializationLibrary=jackson`; forces Java 17; keeps the RFC3339 helpers (shared templates already Jackson 3 ready)
- `microprofile/pom_3.0.mustache`: `tools.jackson.*` groupIds, `jackson-annotations` 2.21, Jackson 3.1.6 (aligned with
  native/apache/jersey3/rest-assured), no `jackson-datatype-jsr310`
- `microprofile/model.mustache`: `{{jacksonPackage}}.dataformat.xml.annotation`
- Tests in `JavaClientCodegenTest` (incl. a check that no generated source references `com.fasterxml.jackson.{core,databind,datatype}`)
- New sample `java/microprofile-rest-client-3.0-jackson3`, built in `samples-java-client-jdk17.yaml`
- Docs regenerated

Verified end to end on a Quarkus 4.0.0.Beta1 application (58 tests, incl. date-time and oneOf/sealed payloads).

### PR checklist
...

@bbdouglas @sreeshas @jfiala @lukoyanov @cbornet @jeff9finger @karismann @Zomzog @lwlee2608 @martin-mfg
```
