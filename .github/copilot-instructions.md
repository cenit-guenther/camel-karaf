# Copilot Instructions — Apache Camel Karaf

This repo repackages Apache Camel (`camel-version` property in root `pom.xml`) as OSGi bundles and Karaf features. It contains almost no Camel logic itself; it is mostly build/packaging metadata plus Pax Exam integration tests.

## Build & test

Use the Maven wrapper (CI uses Java 17 and 21, Maven via `./mvnw`).

```
./mvnw clean install -DskipTests                      # full build (CI step 1)
./mvnw install -Ddump.logs.on.failure=true            # full build + tests (CI step 2)
./mvnw install -pl components/camel-jms               # build one component bundle
./mvnw install -pl tests/features/camel-jms           # run one feature integration test
./mvnw install -pl tests/features/camel-jms -Dtest=CamelJmsITest   # single test class
```

- Feature integration tests (`tests/features/*`) boot a real Karaf via Pax Exam and use Testcontainers (Docker required) for external services. Tests run through surefire (see `tests/features/pom.xml`); `-Ddump.logs.on.failure=true` and `-Dkeep.docker.images.on.exit` are supported.
- Run `install` (not just `test`) when changing bundles/features, since itests resolve `mvn:` URLs from the local repo.
- There is no separate lint step.

## Architecture

Root modules: `core`, `shell`, `components`, `features`, `tests`, `tooling` (`archetypes`/`bom` are commented out).

- **`core/*`** – OSGi-ified Camel core modules (camel-api, camel-base, camel-core-osgi, camel-main, …).
- **`components/camel-<name>`** (~317) – each is a pom-only module (`packaging=bundle`) that depends on the upstream `org.apache.camel:camel-<name>` with all `org.apache.camel:*` transitive deps excluded, then uses maven-shade-plugin to include only that artifact, and sets `camel.osgi.export` / `camel.osgi.import` properties for bnd. It also pulls `../../src/main/resources` (shared LICENSE/NOTICE) into `META-INF`.
- **`features/src/main/feature/camel-features.xml`** (and `spring-features.xml`) – the single Karaf features repository. Each `<feature name="camel-xxx">` references `camel-core`, third-party feature/bundle dependencies (`dependency='true'` for transitive bundles), and the `mvn:org.apache.camel.karaf/camel-xxx/${project.version}` bundle.
- **`tests/features/camel-<name>`** – one Pax Exam itest per feature: a test class in `org.apache.karaf.camel.itest` annotated with `@CamelKarafTestHint`, extending `AbstractCamelSingleFeatureResultMockBasedRouteITest` (uses `@RunWith(PaxExamWithExternalResource.class)`), plus a Blueprint route in `src/main/resources/OSGI-INF/blueprint/route.xml`. External services are provided via `externalResourceProvider` (e.g. `GenericContainerResource`).
- **`tests/camel-*`, `tests/examples`** – Karaf-specific test support (blueprint, SCR, junit5) and example apps.
- **`tooling/camel-upgrade`** – tool used to sync with a new Camel release (see README "Upgrade": clone `apache/camel` next to this repo, checkout the target tag, build camel-karaf, then `java -jar tooling/camel-upgrade/target/camel-upgrade-*.jar` and review `git diff`). `tooling/camel-karaf-feature-maven-plugin` and the test-feature archetype support feature/itest generation.
- **`docs/`** – Antora docs (`docs/modules/ROOT/pages/components.adoc` lists components).

## Conventions

- **Adding/changing a component touches several places**: `components/camel-<name>/pom.xml`, the `<module>` entry in `components/pom.xml`, the `<feature>` in `features/src/main/feature/camel-features.xml`, and a matching `tests/features/camel-<name>` module (registered in `tests/features/pom.xml`). Copy an existing sibling (e.g. `camel-jms`) as the template.
- Module versions are the parent's (`4.18.x-SNAPSHOT`); versions of third-party deps come from properties in the root/features poms, not hardcoded per component.
- Bundle OSGi import tweaks (e.g. `!org.apache.camel.component.cron*, *`) live in `camel.osgi.import`; keep exports as `org.apache.camel*;version=${camel-version}`.
- Test modules are named `camel-<name>-test`; test classes `Camel<Name>ITest`.
- Commits use Conventional-style messages referencing issues, e.g. `fix(#707): …`; backports to release branches are noted `(backport of #NNN)`. Source files carry the Apache license header.
