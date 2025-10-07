---
apply: always
---

SwiftLintRedux – Developer Guidelines

Scope
- Audience: experienced IntelliJ Platform plugin developers working on this project.
- Goal: capture project-specific build, testing, and development conventions verified against the current repository state.

1) Build and Configuration
- Toolchain: Kotlin/JVM with JDK 21. The Gradle build config enforces kotlin.jvmToolchain(21). Ensure JAVA_HOME points to JDK 21 when using local Gradle; the wrapper (./gradlew) will use toolchains automatically.
- Build system: Gradle with the IntelliJ Platform Gradle Plugin and version catalogs.
  - Key plugins (see build.gradle.kts):
    - org.jetbrains.kotlin.jvm
    - org.jetbrains.intellij.platform
    - org.jetbrains.changelog
    - org.jetbrains.qodana
    - org.jetbrains.kotlinx.kover
  - IntelliJ Platform dependencies are declared via the intellijPlatform {} block and gradle.properties.
- Required Gradle properties (gradle.properties):
  - pluginGroup, pluginName, pluginVersion
  - platformType, platformVersion
  - pluginSinceBuild
  - platformBundledPlugins/platformBundledModules/platformPlugins as CSV lists when needed
  - pluginRepositoryUrl
  - gradleVersion (used by the wrapper task)
- Repositories: Maven Central + IntelliJ Platform default repositories via intellijPlatform { defaultRepositories() }.
- Signing/Publishing (optional):
  - Signing env vars: CERTIFICATE_CHAIN, PRIVATE_KEY, PRIVATE_KEY_PASSWORD
  - Marketplace token: PUBLISH_TOKEN
  - Release channel is inferred from pluginVersion pre-release suffix (e.g., 1.2.3-alpha.1 → channel "alpha").
- Typical build commands:
  - Clean & build: ./gradlew clean build
  - Assemble & verify plugin: ./gradlew buildPlugin verifyPlugin
  - Run IDE for manual checks: ./gradlew runIde
  - Update wrapper per gradleVersion: ./gradlew wrapper

2) Testing
2.1 Test Framework
- Tests use the IntelliJ Platform test framework via intellijPlatform { testFramework(TestFrameworkType.Platform) }.
- Base class: com.intellij.testFramework.fixtures.BasePlatformTestCase.
- Test data path: annotate classes with @TestDataPath("$CONTENT_ROOT/src/test/testData") and override getTestDataPath() when using test data files.
- Examples in repo: src/test/kotlin/com/github/dnedrow/swiftlintredux/MyPluginTest.kt with rename and XML PSI checks and test data in src/test/testData/rename.

2.2 Running Tests
- All tests: ./gradlew test
- Single test class (FQN): ./gradlew test --tests "com.github.dnedrow.swiftlintredux.MyPluginTest"
- Single test method: ./gradlew test --tests "com.github.dnedrow.swiftlintredux.MyPluginTest.testRename"
- From IDE: use the Gradle tool window or gutter run icons. Ensure the Gradle JDK is set to 21.
- Coverage: ./gradlew koverXmlReport (XML) or rely on onCheck=true configured under kover.reports.total.xml.

2.3 Adding Tests
- Create a new Kotlin test under src/test/kotlin using the same package as production code to access package-private APIs when needed.
- Use BasePlatformTestCase for light fixtures (PSI, VFS, actions). For UI automation, prefer separate UI tests (see 2.5).
- Example patterns:
  - Fixture text file: val psi = myFixture.configureByText(XmlFileType.INSTANCE, "<foo>bar</foo>")
  - Rename test using test data: myFixture.testRename("foo.xml", "foo_after.xml", "newName")
- If using file-based fixtures, place inputs/expected files under src/test/testData/<area> and either set @TestDataPath or override getTestDataPath().

2.4 Minimal Sanity Test Example (verified)
- We created and ran this minimal test locally to verify the setup, then removed it to keep the repository clean. You can recreate it if you need a template:

  File: src/test/kotlin/com/github/dnedrow/swiftlintredux/SimpleSanityTest.kt
  ---
  package com.github.dnedrow.swiftlintredux

  import com.intellij.testFramework.fixtures.BasePlatformTestCase

  class SimpleSanityTest : BasePlatformTestCase() {
      fun testBundleMessageAvailable() {
          val msg = MyBundle.message("shuffle")
          assertEquals("Shuffle", msg)
      }
  }
  ---

- Run just this test:
  ./gradlew test --tests "com.github.dnedrow.swiftlintredux.SimpleSanityTest"

- Expected: test passes. Note: During tests you may see a warning like:
  "Configuration file for j.u.l.LogManager does not exist: .../test-log.properties" — this is a benign IntelliJ test harness warning and doesn’t indicate a failure.

2.5 UI Test Harness
- A runIde variant for UI tests is preconfigured:
  - Task: runIdeForUiTests registered under intellijPlatformTesting.runIde.
  - It injects robot-server args and applies the Robot Server plugin automatically.
- Launch for UI testing:
  ./gradlew runIdeForUiTests
- CI workflows include UI test steps; see .github/workflows/run-ui-tests.yml.

3) Additional Development Notes
- Code style:
  - Kotlin (no ktlint rules configured in this repo). Follow standard Kotlin style with explicit visibility modifiers as appropriate.
  - JVM target aligns with toolchain 21; avoid APIs not available under the target IDE baselines.
- Internationalization:
  - Messages are in src/main/resources/messages/MyBundle.properties and retrieved via MyBundle.message(key).
- Logging:
  - Use com.intellij.openapi.diagnostic.Logger (thisLogger()) for component/activity logs. Sample code logs a reminder to remove sample scaffolding.
- Extension points & registration:
  - Check src/main/resources/META-INF/plugin.xml for components, actions, services, and extensions. Keep registrations in sync with source files; remove sample entries if you delete template code.
- Running the plugin for manual checks:
  - ./gradlew runIde starts a sandbox IDE with the plugin.
  - For consistent results, clear .gradle and build/ caches only if necessary; sandbox is under build/idea-sandbox.
- Changelog & plugin description:
  - The Gradle build extracts plugin description from README.md between markers and change notes from CHANGELOG.md via the Changelog plugin. Keep those files in sync with releases.
- Static analysis:
  - Qodana is configured via qodana.yml. Run locally with Docker or the Qodana Gradle task if desired.
- CI/CD:
  - GitHub Actions workflows present: build.yml, release.yml, run-ui-tests.yml; Dependabot is enabled.
- Git Commit Messages:
  - Always use Conventional Commits (https://www.conventionalcommits.org/en/v1.0.0/) for commit messages.

4) Quickstart Commands
- Build: ./gradlew build
- Run IDE: ./gradlew runIde
- Tests: ./gradlew test
- One test: ./gradlew test --tests "com.github.dnedrow.swiftlintredux.MyPluginTest.testXMLFile"
- Plugin verification (against recommended IDEs): part of intellijPlatform.pluginVerification; run ./gradlew verifyPlugin

Housekeeping
- Temporary artifacts used to validate these guidelines (e.g., SimpleSanityTest.kt) were created and removed during authoring so the repository stays unchanged except for this guidance file.
- If you introduce new sample/tests, clean up after demonstration or clearly mark them as permanent.
