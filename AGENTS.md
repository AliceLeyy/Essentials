# AGENTS.md

EssentialsX: a multi-module Gradle plugin suite for Bukkit/Spigot/Paper. Main branch is `2.x`.

## Build & run

- `./gradlew build` — compiles every module and copies plugin jars into `jars/` (NOT the usual
  `build/libs`). The `copyJar` task wires this into `build`.
- `./gradlew build :runServer` — run a Paper test server. **The leading `:` matters**: without it Gradle
  starts one server per module and likely crashes. Server dir `run/` is gitignored.
  - Add modules with `-PrunModules=chat,spawn`; run everything with `./gradlew runAll`.
- `./gradlew test` — JUnit 5 tests. Only `Essentials` has tests (11 files), all MockBukkit-based.
- Build needs JDK 25 (Indra `minimumToolchain`); CI uses Temurin 25. Compiled bytecode still targets
  Java 8 (`target(8)`), so no Java 9+ APIs.
- `./gradlew :EssentialsX:compileJava` — name of the core module; useful to force-download the Paper API.
- Checkstyle runs as part of `check`/`build` (Indra). Config: `.checkstyle/checkstyle.xml`
  (tabs forbidden, `FinalLocalVariable`, `RequireThis`, `UnusedImports`, newline at EOF, etc.).

## Module map

Gradle project names differ from directories:
`:EssentialsX`→`Essentials`, `:EssentialsXChat`→`EssentialsChat`, plus `AntiBuild`, `Discord`,
`DiscordLink`, `GeoIP`, `Protect`, `Spawn`, `XMPP`. Core code is `Essentials/src/main/java/com/earth2me/essentials`.

`providers/` holds compatibility providers, each compiled against a specific old API:
`BaseProviders` (modern, spigot-api 1.21.5), `PaperProvider`, `NMSReflectionProvider` (bukkit 1.12.2),
`1_12Provider`, `1_8Provider`. Providers are selected at runtime by `ProviderFactory` from annotations
(`@ProviderData(weight=...)`, `@ProviderTest`) and registered in `Essentials.java`. To change behavior on
old servers, add/lower a provider here rather than branching in core code.

## Multi-version compatibility (the core constraint)

Code compiles against a very new Paper API but must run on 1.8.8+. This drives most rules:

- Never reference a new `Material`/`EntityType`/`Enchantment`/`PotionEffectType` constant in a static
  field or enum initializer — use `EnumUtil`, `RegistryUtil`, or name lookups instead.
- Catch `Throwable` (not `Exception`) around reflection/provider selection.
- Only Guava/Gson APIs present in 1.8.8.
- Gate with `VersionUtil.isLowerThan(...)`; never delete a `VersionUtil` constant. Constants and
  `supportedVersions` live in `Essentials/src/main/java/com/earth2me/essentials/utils/VersionUtil.java`
  (`supportedVersions` must keep `ImmutableSet.of(...)` syntax — `VersionDataTask` regex-parses it).
- New API methods may need stubs in `providers/1_8Provider/.../OfflinePlayer.java` or
  `providers/1_12Provider/.../FakeWorld.java` to compile.

For a Minecraft/Paper version bump, load the `mc-update` skill
(`.claude/skills/mc-update/SKILL.md`) — it is the checklist of everything that must be touched and the
compatibility traps from past updates. Set `paperVersion` in
`build-logic/src/main/kotlin/essentials.base-conventions.gradle.kts`.

## Tests vs. compile target

`paperVersion` (compile target, currently 26.3-pre) and `paperTestVersion` (1.21.8, with MockBukkit
v1.21) differ in `essentials.base-conventions.gradle.kts` — tests run against the older Paper API on
purpose. Don't "fix" the mismatch.

## Generated data & translations

- `./gradlew commandData versionData` writes command/permission/version JSON into `generated/`
  (gitignored, uploaded by CI). `CommandDataTask` harvests `plugin.yml` + regex-matches
  `*Command(Description|Usage)N` keys in `Essentials/src/main/resources/messages.properties`.
- Only edit the English source `messages.properties`. Other `messages_*.properties` are Crowdin-managed
  (`crowdin.yml`); don't hand-edit translations.

## Conventions

- Feature PRs need prior discussion in a GitHub issue/discussion; bug-fix PRs need an existing report
  (see `CONTRIBUTING.md`). CI builds on `2.x`, `mc/*`, `pr/*`, `dev/*`.
- Do not commit unless asked.
