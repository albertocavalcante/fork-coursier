# Investigation: GMM Variant Resolution Missing Classifier in Coordinates

**Date**: 2026-01-20
**Status**: Root Cause Identified
**Severity**: High (breaks compilation for projects using GMM + fetch_sources)

---

## 1. Executive Summary

When Coursier resolves dependencies using Gradle Module Metadata (GMM) and `fetch_sources=true`, it correctly fetches both main JARs and sources JARs from their respective variants. However, it generates **identical coordinates** for both artifacts in the JSON report, causing downstream tools to potentially use sources JARs instead of main JARs.

**Root Cause**: `VariantPublication` does not carry classifier information, and variant attributes (like `org.gradle.docstype=sources`) are lost during processing.

---

## 2. Symptom

### 2.1 Build Failure

```bash
$ bazel build //...
error: unresolved reference 'ModuleNode'
```

### 2.2 Generated BUILD File Shows Wrong JAR

```starlark
jvm_import(
    name = "org_apache_groovy_groovy",
    jar = "v1/.../groovy-4.0.29-sources.jar",  # WRONG!
    # Should be: groovy-4.0.29.jar
)
```

### 2.3 Scale

~265 artifacts affected in the test project.

---

## 3. Evidence: The dep-tree.json Difference

### 3.1 GMM Artifacts (BUG)

Same coordinate, different files:
```json
{"coord":"androidx.annotation:annotation-jvm:1.8.0","file":"...annotation-jvm-1.8.0.jar"},
{"coord":"androidx.annotation:annotation-jvm:1.8.0","file":"...annotation-jvm-1.8.0-sources.jar"}
```

### 3.2 Non-GMM Artifacts (CORRECT)

Classifier in coordinate:
```json
{"coord":"aopalliance:aopalliance:1.0","file":"...aopalliance-1.0.jar"},
{"coord":"aopalliance:aopalliance:jar:sources:1.0","file":"...aopalliance-1.0-sources.jar"}
```

---

## 4. GMM Files Are Correctly Structured

Investigation confirmed that GMM files from publishers (AndroidX, Groovy, etc.) are **correctly structured**:

### 4.1 AndroidX Collection Example

```json
{
  "variants": [
    {
      "name": "jvmApiElements-published",
      "attributes": {
        "org.gradle.category": "library",
        "org.gradle.usage": "java-api"
      },
      "files": [{"name": "collection-jvm-1.4.0.jar", ...}]
    },
    {
      "name": "jvmSourcesElements-published",
      "attributes": {
        "org.gradle.category": "documentation",
        "org.gradle.docstype": "sources"
      },
      "files": [{"name": "collection-jvm-1.4.0-sources.jar", ...}]
    }
  ]
}
```

**Conclusion**: The GMM files correctly separate main JARs and sources JARs into different variants with appropriate attributes. The bug is NOT in the GMM files.

---

## 5. Root Cause Analysis

### 5.1 Data Flow

```
GMM File
    ↓
GradleModule.scala: Creates variantPublications Map[Variant.Attributes, Seq[VariantPublication]]
    ↓ (Variant.Attributes LOST here)
MavenRepositoryInternal.scala: Returns Seq[(VariantPublication, Artifact)]
    ↓
Resolution.scala: dependencyArtifacts0() returns Seq[(Dependency, Either[VariantPublication, Publication], Artifact)]
    ↓
JsonReport.scala: Generates coordinates - BUT VariantPublication has no classifier!
```

### 5.2 The VariantPublication Data Class

**File**: `modules/core/shared/src/main/scala/coursier/core/Definitions.scala`

```scala
@data class VariantPublication(
  name: String,
  url: String
)
```

**Problem**: Only has `name` and `url`. No classifier, no variant attributes.

### 5.3 The Coordinate Generation

**File**: `modules/cli/src/main/scala/coursier/cli/fetch/JsonReport.scala`
**Lines**: 234-235

```scala
val attr = pub.fold(_ => dep, pub0 => dep.withPublication(pub0)).attributes
```

For `VariantPublication` (Left/GMM case): Uses `dep.attributes` which has NO classifier.
For `Publication` (Right/Maven case): Uses `dep.withPublication(pub0).attributes` which HAS classifier.

### 5.4 Where Variant Attributes Are Lost

**File**: `modules/core/shared/src/main/scala/coursier/maven/GradleModule.scala`
**Lines**: 154-161

```scala
val variantPublications = variants
  .map { variant =>
    val publications = variant.files.map { file =>
      VariantPublication(file.name, file.url)  // <-- No attributes!
    }
    Variant.Attributes(variant.name) -> publications  // <-- Attributes in key, not in VariantPublication
  }
  .toMap
```

The variant attributes (including `org.gradle.category=documentation`, `org.gradle.docstype=sources`) are in the map KEY (`Variant.Attributes`), but not in the VALUE (`VariantPublication`). When `VariantPublication` is extracted from the map and passed down the chain, the attribute information is lost.

---

## 6. GMM Specification Analysis

### 6.1 Key Attributes

| Attribute | Purpose | Values |
|-----------|---------|--------|
| `org.gradle.category` | Component type | `library`, `platform`, `documentation` |
| `org.gradle.docstype` | Documentation type | `sources`, `javadoc`, `groovydoc`, `doxygen` |
| `org.gradle.usage` | API vs Runtime | `java-api`, `java-runtime` |

### 6.2 Relevant Spec Quote

> The `org.gradle.category` attribute "is mostly used to disambiguate Maven POM files derived either as a platform or a library."

> `org.gradle.docstype` applies to variants where `org.gradle.category=documentation`. Valid values: `javadoc`, `sources`, `doxygen`.

---

## 7. Proposed Fix Options

### Option A: Enrich VariantPublication (Recommended)

Add classifier to `VariantPublication`:

```scala
@data class VariantPublication(
  name: String,
  url: String,
  classifier: Option[Classifier] = None  // NEW
)
```

In `GradleModule.scala`, derive classifier from variant attributes:

```scala
val variantPublications = variants
  .map { variant =>
    val isDocumentation = variant.attributesMap.get("org.gradle.category").contains("documentation")
    val docsType = variant.attributesMap.get("org.gradle.docstype")
    val classifier = if (isDocumentation) docsType.map(Classifier(_)) else None

    val publications = variant.files.map { file =>
      VariantPublication(file.name, file.url, classifier)
    }
    Variant.Attributes(variant.name) -> publications
  }
  .toMap
```

Then update `JsonReport.scala` to use `VariantPublication.classifier` when generating coordinates.

### Option B: Infer Classifier from File Name (Fallback)

In `JsonReport.scala`, detect `-sources.jar` or `-javadoc.jar` suffixes and add classifier:

```scala
val inferredClassifier =
  if (fileName.endsWith("-sources.jar")) Some("sources")
  else if (fileName.endsWith("-javadoc.jar")) Some("javadoc")
  else None
```

**Pros**: Simple, no core type changes.
**Cons**: Relies on naming convention, not spec-compliant.

### Option C: Propagate Variant.Attributes Through Chain

Pass `Variant.Attributes` alongside `VariantPublication` through the entire processing chain.

**Pros**: Most complete solution.
**Cons**: Large refactor, touches many files.

---

## 8. Files to Modify

| File | Change |
|------|--------|
| `modules/core/shared/src/main/scala/coursier/core/Definitions.scala` | Add `classifier` to `VariantPublication` |
| `modules/core/shared/src/main/scala/coursier/maven/GradleModule.scala` | Derive classifier from variant attributes |
| `modules/cli/src/main/scala/coursier/cli/fetch/JsonReport.scala` | Use `VariantPublication.classifier` in coordinate generation |
| `modules/cli/src/test/scala/coursier/cli/JsonReportTests.scala` | Add tests for GMM classifier handling |

---

## 9. TDD Test Cases

### 9.1 Unit Tests for GradleModule

1. **Test**: Documentation variant with `docstype=sources` → `VariantPublication.classifier = Some("sources")`
2. **Test**: Documentation variant with `docstype=javadoc` → `VariantPublication.classifier = Some("javadoc")`
3. **Test**: Library variant → `VariantPublication.classifier = None`
4. **Test**: Variant with `category=documentation` but no `docstype` → `VariantPublication.classifier = None`

### 9.2 Integration Tests for JsonReport

5. **Test**: GMM artifact with sources → coordinate includes `:sources` classifier
6. **Test**: GMM artifact without sources → coordinate has no classifier
7. **Test**: Multiple files from same dependency (main + sources) → different coordinates

### 9.3 End-to-End Tests

8. **Test**: Resolve dependency with GMM + `fetch_sources=true` → dep-tree.json has correct classifiers

---

## 10. Existing Tests Analysis

### 10.1 JsonReportTests.scala

**Location**: `modules/cli/src/test/scala/coursier/cli/JsonReportTests.scala`

**Pattern**: Uses golden file comparison
- `doCheck()` runs fetch and compares JSON output to files in `test-data/reports/`
- Has tests for sources with non-GMM artifacts (line 299-304)
- Has tests for Android/GMM artifacts but WITHOUT sources

**Gap**: No tests for GMM artifacts + sources combined.

### 10.2 FetchTests.scala (GMM + Sources)

**Location**: `modules/coursier/jvm/src/test/scala/coursier/tests/FetchTests.scala`

**Lines 619-680**: Tests for GMM + sources fetching exist!
```scala
test("sources") {
  test("compile") {
    val classifiers = Set(Classifier.sources)
    val attr = Seq(VariantSelector.AttributesBased.sources)
    // Fetches kotlin-stdlib sources via GMM
  }
  test("default") {
    // Fetches material3 sources via GMM
  }
}
```

**What these tests verify**: Artifact resolution works correctly.
**What they DON'T verify**: JSON report coordinates.

### 10.3 ResolveTests.scala (GMM Resolution)

**Location**: `modules/coursier/shared/src/test/scala/coursier/tests/ResolveTests.scala`

**Lines 2063-2376**: Comprehensive GMM resolution tests
- `gradleModuleCheck()` helper function
- Tests for Kotlin, AndroidX, Compose artifacts
- Tests various variant attribute combinations

**Gap**: None of these tests fetch sources.

### 10.4 Key Testing Pattern

```scala
// Enable GMM support
def enableModules(resolve: Resolve[Task]) =
  resolve.mapResolutionParams(_.withGradleModulesSupport(true))

// For GMM, use variant attributes
fetch.withArtifactAttributes(Seq(VariantSelector.AttributesBased.sources))

// Golden file validation
TestHelpers.validateResult("path/to/expected.json") {
  jsonLines(JsonReport.report(res.resolution, res.fullDetailedArtifacts0))
}
```

### 10.5 Missing Test Case

We need a test that:
1. Enables GMM support
2. Fetches a dependency with both main artifacts AND sources
3. Validates the JSON report has **different coordinates** for main vs sources

---

## 11. Next Steps

1. [x] Identify root cause
2. [x] Document findings
3. [ ] Study existing tests in `JsonReportTests.scala`
4. [ ] Write failing tests (TDD)
5. [ ] Implement Option A fix
6. [ ] Verify fix doesn't break existing behavior
7. [ ] Submit PR to upstream Coursier

---

## 12. Contributing to Coursier

### 12.1 Repository

- Upstream: https://github.com/coursier/coursier
- Fork: https://github.com/albertocavalcante/fork-coursier

### 12.2 Build & Test

```bash
# Build all
./mill -i __.compile

# Run all tests
./mill -i __.test

# Run specific test suite
./mill -i cli.test.testOnly coursier.cli.JsonReportTests

# Run specific test
./mill -i cli.test.testOnly coursier.cli.JsonReportTests -- "test name pattern"
```

### 12.3 Contribution Guidelines

See: https://github.com/coursier/coursier/blob/main/CONTRIBUTING.md

---

## 13. Concrete Example: kotlinx-serialization-json-jvm

### 13.1 The GMM File (Correctly Structured)

**File**: `kotlinx-serialization-json-jvm-1.9.0.module`

```json
{
  "variants": [
    {
      "name": "jvmRuntimeElements-published",
      "attributes": {
        "org.gradle.category": "library",        // <-- LIBRARY variant
        "org.gradle.usage": "java-runtime"
      },
      "files": [
        {"name": "kotlinx-serialization-json-jvm-1.9.0.jar", ...}
      ]
    },
    {
      "name": "jvmSourcesElements-published",
      "attributes": {
        "org.gradle.category": "documentation",   // <-- DOCUMENTATION variant
        "org.gradle.docstype": "sources"          // <-- SOURCES type
      },
      "files": [
        {"name": "kotlinx-serialization-json-jvm-1.9.0-sources.jar", ...}
      ]
    }
  ]
}
```

**Note**: The GMM file is CORRECT - sources are in a separate variant with proper attributes.

### 13.2 Coursier dep-tree.json Output (BUG)

**GMM Artifact - SAME coord for different files:**
```json
{
  "coord": "org.jetbrains.kotlinx:kotlinx-serialization-json-jvm:1.9.0",
  "file": "...kotlinx-serialization-json-jvm-1.9.0.jar"
}
{
  "coord": "org.jetbrains.kotlinx:kotlinx-serialization-json-jvm:1.9.0",  // SAME!
  "file": "...kotlinx-serialization-json-jvm-1.9.0-sources.jar"           // DIFFERENT!
}
```

**Non-GMM Artifact - Different coords (CORRECT):**
```json
{
  "coord": "aopalliance:aopalliance:1.0",
  "file": "...aopalliance-1.0.jar"
}
{
  "coord": "aopalliance:aopalliance:jar:sources:1.0",  // INCLUDES :sources classifier!
  "file": "...aopalliance-1.0-sources.jar"
}
```

### 13.3 Expected Fix Output

After the fix, Coursier should output:
```json
{
  "coord": "org.jetbrains.kotlinx:kotlinx-serialization-json-jvm:1.9.0",
  "file": "...kotlinx-serialization-json-jvm-1.9.0.jar"
}
{
  "coord": "org.jetbrains.kotlinx:kotlinx-serialization-json-jvm:jar:sources:1.9.0",  // WITH classifier!
  "file": "...kotlinx-serialization-json-jvm-1.9.0-sources.jar"
}
```

---

## 14. rules_jvm_external Analysis

### 14.1 Key Finding: Coursier Fix is Sufficient

Investigation of `/Users/adsc/dev/forks/fork-rules_jvm_external` confirms:

**rules_jvm_external already handles classifiers correctly** - no changes needed there.

### 14.2 How rules_jvm_external Processes dep-tree.json

1. **Deduplication**: Uses `file` path as key (not `coord`)
   - Both entries survive because they have different files

2. **Target Generation**: Uses `strip_packaging_and_classifier_and_version(coord)`
   - If coords are SAME → same target label → second is SKIPPED
   - If coords are DIFFERENT → different target labels → both processed

3. **Current Bug Flow**:
   ```
   Entry 1: coord="g:a:1.0", file="a.jar"       → target: g_a
   Entry 2: coord="g:a:1.0", file="a-sources.jar" → target: g_a (SAME!)
   → Entry 2 is SKIPPED because target already exists
   → Which entry wins depends on order (usually sources wins = BUG)
   ```

4. **After Coursier Fix**:
   ```
   Entry 1: coord="g:a:1.0", file="a.jar"              → target: g_a
   Entry 2: coord="g:a:jar:sources:1.0", file="a-sources.jar" → target: g_a_jar_sources
   → BOTH entries get different targets
   → Works correctly!
   ```

### 14.3 Relevant rules_jvm_external Code

| File | Function | Purpose |
|------|----------|---------|
| `private/artifact_utilities.bzl:44-48` | Deduplication | Uses `file` as key |
| `private/dependency_tree_parser.bzl:101-102` | Target label | Uses coord with classifier |
| `private/dependency_tree_parser.bzl:490-492` | Duplicate skip | Skips same target label |
| `private/coursier_utilities.bzl:72-73` | Classifier handling | Preserves non-sources/natives classifiers |

---

## 15. Summary

### The Bug
Coursier's GMM resolver correctly identifies sources from documentation variants, but **loses the classifier information** when creating `VariantPublication` objects. This causes `JsonReport` to generate identical coordinates for main JARs and sources JARs.

### The Fix
1. Add `classifier: Option[Classifier]` to `VariantPublication`
2. In `GradleModule.scala`, derive classifier from variant's `org.gradle.docstype` attribute
3. Update `JsonReport.scala` to include classifier in coordinates for `VariantPublication`

### Impact
- **Coursier-only fix is sufficient** - rules_jvm_external already handles classifiers correctly
- Affects ~265 artifacts in typical Kotlin/Android projects
- Currently causes compilation failures when sources JARs are used instead of main JARs

---

## 16. Appendix: Key Code References

### 16.1 VariantPublication Definition
`modules/core/shared/src/main/scala/coursier/core/Definitions.scala:VariantPublication`

### 16.2 GradleModule variant processing
`modules/core/shared/src/main/scala/coursier/maven/GradleModule.scala:154-161`

### 16.3 JsonReport coordinate generation
`modules/cli/src/main/scala/coursier/cli/fetch/JsonReport.scala:190-196` (coords function)
`modules/cli/src/main/scala/coursier/cli/fetch/JsonReport.scala:231-266` (multi-file handling)

### 16.4 VariantSelector sources definition
`modules/core/shared/src/main/scala/coursier/core/VariantSelector.scala:141-151`
