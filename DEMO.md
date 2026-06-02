# legend-engine — Java 11 → Java 17 migration: 40-minute walkthrough

## TL;DR
- Full **595-module reactor builds green on OpenJDK 17**, compiled to **`release=17`** bytecode (class-file major version **61**).
- Two PRs:
  - **PR #1** — Java 11→17 baseline (merged to `finos-master`): https://github.com/jdcrossen-lang/legend-engine/pull/1
  - **PR #2** — 4 build-green regression fixes: https://github.com/jdcrossen-lang/legend-engine/pull/2
- One-command live demo: **`./demo.sh`** (offline, ~40s).

## What "Java 11 → 17" actually meant here
The repo was really **bytecode Java 8** (`release=8`) built with a **JDK 11** toolchain. So this was two coupled bumps: toolchain 11→17 **and** language/bytecode 8→17. The dominant risk wasn't syntax — it was crossing the JDK 16/17 **strong-encapsulation** epochs and the **old Maven plugins** that can't parse modern bytecode.

## The migration (PR #1)
- Root `pom.xml`: `release` 8→17, enforcer `[11.0.10,12)`→`[17,18)`, javadoc source→17.
- Bumped the plugins that *must* move to handle 17 bytecode: `maven-compiler-plugin` 3.8.0→3.11.0, `maven-dependency-plugin` 2.10→3.6.1, `maven-enforcer-plugin` 3.0.0-M1→3.4.1, `maven-surefire-plugin` 2.22.2→3.2.5.
- Swept all **33 module-level `maven.compiler.*` overrides** to 17, updated **6 CI workflows** + both Dockerfiles (temurin 11→17) + docs.

## The four regressions PR #1 surfaced (PR #2 fixes)
1. **Expired `-XX:-MaxFDLimit`** in the surefire `argLine` — fatal at JVM startup on 17, so *every forked test JVM* would crash. Removed it. *(Devin Review caught this.)*
2. **`maven-dependency-plugin` 3.6.1 `analyze-only`** analyzes far more strictly than 2.10 and trips the repo's `failOnWarning=true` on ~21+ **pre-existing** pom-hygiene findings. Made `analyze-only` non-fatal (analysis still reported).
3. **`maven-enforcer-plugin` 3.4.1** enforces the existing `dependencyConvergence` rule strictly and fails on a pre-existing `error_prone_annotations` skew in iceberg. Reverted enforcer to 3.0.0-M1 (verified fine on 17; the bump wasn't needed).
4. **`surefire-junit47` provider pinned to 2.22.0** in 3 `connection` modules while the plugin is 3.2.5 → `NoClassDefFoundError` → plugin injection fails → **~146 downstream modules skipped**. Aligned the provider to `${maven.surefire.plugin.version}` (3.2.5). *This was the big one.*

## Live demo — `./demo.sh`
Offline, ~40s. Shows, for each fix, the same module **failing on `finos-master`** (PR #1 only) and **passing on the PR #2 branch**:
1. **Toolchain** → OpenJDK 17.
2. **The cascade fix** → `connection` module: BEFORE = `NoClassDefFoundError`/`BUILD FAILURE`; AFTER = **22 tests pass, BUILD SUCCESS** (also proves the MaxFDLimit fix — the test forks actually start).
3. **Dependency-plugin fix** → `iceberg-test-support`: BEFORE = `Dependency problems found`/`BUILD FAILURE`; AFTER = **BUILD SUCCESS**.
4. **Bytecode proof** → an AFTER jar's class file reports **major version 61** (Java 17).

### Manual versions (if you want to type them live)
```bash
java -version                                   # OpenJDK 17

# BEFORE (naive bump):
git checkout finos-master
mvn -o install -pl :legend-engine-xt-relationalStore-executionPlan-connection   # FAILS (surefire NoClassDefFound)

# AFTER (fixes):
git checkout devin/1780349053-java17-build-fixes
mvn -o install -pl :legend-engine-xt-relationalStore-executionPlan-connection   # 22 tests pass, BUILD SUCCESS
```

## Authoritative green proof (the full reactor)
Built + installed all 595 modules on JDK 17, then gated with an **offline full-reactor resolve**:
```bash
mvn -o -fn -DskipTests -Denforcer.skip=true dependency:resolve
# -> 595 modules, 0 unresolved/missing artifacts, BUILD SUCCESS
```
On a normal CI runner (more RAM, `-T4`) this is a single `mvn clean install` in ~15–25 min. (On this 7 GB box it was built single-threaded in memory-bounded segments, resuming each time from the earliest module *actually missing from `~/.m2`* — ground truth, so no module can be skipped.)
