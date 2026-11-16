# Triweave 0.1.0

A Fabric-based mod loader for Minecraft Java **26.3**.
It sits on top of Fabric Loader (>=0.19.3), translates Quilt mods, and runs NeoForge mods and
Bukkit-family plugins (plugin.yml format) through limited compatibility layers.

Languages: [日本語](README.md) / English (this file) / [中文](README.zh.md)

## What it does

At startup (just before Fabric itself starts), it scans `mods/` and decides how to run each jar.

|Jar contents|How it runs|
|-|-|
|Has `fabric.mod.json` (even alongside Quilt/NeoForge metadata)|**Runs as Fabric, unchanged** (highest priority)|
|Only `quilt.mod.json` (even alongside NeoForge metadata)|**Translated to Fabric format and run** (next priority)|
|Only `neoforge.mods.toml`|Evaluated against the compatibility layer; accepted and run, or rejected with a reason (see below)|
|Only `mods.toml` (legacy Forge)|Reported as unsupported|

* Fabric API (mod jar, 0.166.0+26.3) is bundled and auto-installed into `mods/` if missing.
* Original jars are never modified. Translated copies go into `.triweave/cache/` and are reused by
SHA-1 (so later launches are fast).
* If there are no Quilt mods, no translation work runs at all.

## triweave.mod.json (Triweave's own format)

Triweave-specific metadata. Load priority is **`triweave.mod.json` > `fabric.mod.json` >
`quilt.mod.json` > `neoforge.mods.toml`** (a jar that also ships `fabric.mod.json` is still loaded
directly by Fabric as a Fabric mod — that part of Fabric's behavior can't be changed — but
`triweave.mod.json` is still used for the display name/version shown to the user). Fabric, Quilt
and NeoForge jars keep working exactly as before. Format: `docs/triweave.mod.json.md`. Example:
`examples/triweave.mod.json`.

## Triweave API (loader-independent)

Cross-loader mod info, enable/disable, reload hooks, a service registry, simple config files, and
more. Details in `docs/API.md`. A standalone `triweave-api-0.1.0.jar` (classes + source) is
provided for mod developers.

## Scope of Quilt translation

* `quilt.mod.json` → `fabric.mod.json` (metadata, dependencies, provides, mixins, entrypoints,
jar-in-jar, environment).
* Dependency remapping: `quilt\_loader` → `fabricloader`, `quilted\_fabric\_api` → `fabric-api`.
* QSL module dependencies (`quilt\_base` etc.) are dropped (with a warning).
* Entrypoints go through the `triweave-quilt` language adapter, which calls QSL-style
`onInitialize(ModContainer)` / `onInitializeClient` / `onInitializeServer` / `onPreLaunch` as if
they were Fabric types.
* A minimal shim of the Quilt API (`QuiltLoader`, `ModContainer`, `ModMetadata`, `Version`) is
bundled.
* Not supported: `Class::method`-style entrypoints, the real QSL API surface (mods that use large
Quilt-only APIs), and multiple access wideners (only the first is used).

## Mod Menu listing for NeoForge mods

* Mod Menu (21.0.0) is bundled the same way as Fabric API, and auto-installed into `mods/` if
missing.
* NeoForge-only mods get a **Fabric-format listing stub** (`nf\_<modid>`, name suffixed with
`\[NeoForge]`) so they show up in Mod Menu. The stub only contains `fabric.mod.json` and an icon —
**the mod's actual functionality does not run**. The `nf\_` prefix means it can never accidentally
satisfy a real Fabric mod's dependency, and it never collides with a same-named Fabric mod (no
stub is generated if a Fabric/Quilt mod with the same name already exists).
* If multiple jars declare the same mod id, Fabric refuses to start, so Triweave logs
`DUPLICATE mod id` to flag this ahead of time.

## How NeoForge mods are handled

At startup, each NeoForge-only jar is mechanically evaluated for whether "Triweave's compatibility
layer can run it".

**Accepted (runs)**: mods that only use NeoForge APIs the compatibility layer actually provides —
mainly MCreator-generated mods adding blocks / items / creative tabs / recipes / biomes /
dimensions, etc. Triweave adds `fabric.mod.json` and points Fabric's main entrypoint at the `@Mod`
class (since 26.x uses Mojang mappings for both Fabric and NeoForge, no code rewriting is needed).

**Rejected (does not run, reason shown)**:

* Network messaging (a real message class using `CustomPacketPayload`)
* Kotlin (`modLoader="klf"`, a KotlinForForge dependency, or Kotlin bytecode)
* NeoForge-style Mixins or AccessTransformers
* Use of any NeoForge API the compatibility layer doesn't provide (the missing API names are shown)

Rejected mods only get a Mod Menu listing stub (`nf\_<modid>`); nothing runs.
Run `java -jar triweave-0.1.0.jar check <jar...>` to see the verdict and reasons ahead of time.

What the compatibility layer provides: `@Mod` / constructor parameters (IEventBus, ModContainer,
Dist), `@EventBusSubscriber` / `@SubscribeEvent`, `DeferredRegister` (Items, Blocks, generic),
`RegisterEvent`, `BuildCreativeModeTabContentsEvent` (wired to Fabric API's creative-tab events),
`ServerTickEvent`, `FMLCommonSetupEvent`, server start/stop events, player login/logout,
`ClientTickEvent`, and **`BlockEvent.BreakEvent` (block breaking, cancellable) and
`LivingDeathEvent` (death, cancellable)**. Any other event a mod listens for is logged as "never
fired" so the mod author knows that handler won't run.

Known limitations: `DeferredHolder` does not implement `Holder`; `SidedThreadGroups.SERVER`
detection always returns false (so `queueServerWork`'s deferred execution doesn't run); GUIs
(menu screens) aren't implemented; in-world gameplay events beyond the ones listed above aren't
wired up. **Real-hardware behavior is unverified.**

## Modrinth App and other launchers that can't swap the main class

Just add this to the JVM arguments and preparation happens automatically on every launch:

```
-javaagent:C:\\Triweave\\triweave-0.1.0.jar
```

Create a Fabric 26.3 instance and add the argument above. Right before Fabric starts, it writes
translated jars and the Triweave core mod into `mods/` with a `\_\_triweave\_` prefix (removing a mod
cleans up its generated copy on the next launch automatically). Manual run:
`java -jar triweave-0.1.0.jar prepare --gameDir <instance folder>`

## In-game commands

|Command|What it does|
|-|-|
|`/mod`|Full list of Fabric / Quilt / NeoForge mods|
|`/mod <kind> \[list]`|List for one kind: `triweave` / `fabric` / `quilt` / `neoforge` / `bukkit`|
|`/mod <name> \[info]`|Mod details (id, version, loader, state, file)|
|`/mod <name> on` / `off`|Enable/disable a mod (**takes effect on next launch**; on a server this needs OP; Bukkit plugins can't be toggled this way)|
|`/mod <name> reload`|Runs the mod's registered reload hooks + reloads its config + reloads resources|
|`/mod triweave \[reload]`|Triweave info (Discord link) / reload everything|

* Mod names can be matched by id, display name, or partial match (quote names containing spaces).
* "Off" is implemented by renaming the jar to `\*.jar.triweave-off` before launch (distinct from
Modrinth's `.disabled` convention). A mod required by another mod, `fabric-api`, or `triweave`
itself cannot be turned off.
* **Code cannot be hot-reloaded** (a JVM limitation). Adding/removing mods or toggling them only
takes effect after a restart.
* The listing is written at startup to `.triweave/modlist.tsv`. The command itself exists on both
the client and the server.

## Launch modes (JVM argument)

* Default: writes translated jars into `mods/` with a `\_\_triweave\_` prefix (verified on real
hardware).
* `-Dtriweave.mode=addmods`: writes nothing into `mods/`; instead passes jars to Fabric via
`fabric.addMods` (**unverified**).

## Support / bug reports

Discord: https://discord.gg/ZqDd74CC8B (also reachable from Mod Menu's "Website"/"Report bug"
links)

> Note: the `/mod` command and the NeoForge compatibility layer depend on Minecraft's classes, so
> `gradle build` will not work with this Gradle setup as-is (it would need something like Loom to
> add Minecraft as a dependency, which hasn't been set up/verified for 26.3).
> `tools/stubs-and-compile.sh` builds `main` against compile-only stubs instead (the API list is
> regenerated with `tools/GenApiList.java`). The distributed `triweave-0.1.0.jar` was compiled
> against the real method signatures of the 26.3 Fabric API.

## Distributed files (separate client / server builds)

|File|Contents|
|-|-|
|`triweave.cliant.26.3.0.1.0.jar`|Client launcher. Only `TriweaveClient` / the client-side `/mod` command; bundles Mod Menu|
|`triweave.server.26.3.0.1.0.jar`|Server launcher. Only `TriweaveServer` / the server-side `/mod` command; does not bundle Mod Menu|
|`triweave.api.26.3.0.1.0.jar`|`dev.triweave.api` standalone (classes + source), for mod developers|
|`triweave.core.mod.fabric.26.3.0.1.0.jar`|The plain Fabric mod itself, not wrapped in the launcher (contains both client- and server-side entrypoints). For adding directly as a mod to an existing Fabric instance. However, Quilt translation, NeoForge detection, and the `/mod` listing (`modlist.tsv`) are produced by the pre-launch step (the launcher itself, or `-javaagent`) — so dropping this alone into `mods/` will not give you those features|
|`triweave.source.zip`|Full source (`README.md`, `docs/`, `examples/`)|

## Building

Requires JDK 21+ and Gradle 8.x (the game itself needs Java 25 at runtime).

```
gradle build
```

`build/libs/triweave-0.1.0.jar` is the distributable (it embeds the core mod jar and Fabric API).
To swap the bundled Fabric API, replace `src/launcher/resources/triweave/fabric-api.jar` and update
`bundled.properties`.

### About the bundled Fabric API / Mod Menu

This build bundles **Fabric API 0.166.0+26.3** and **Mod Menu 21.0.0**. As far as could be
determined, Mod Menu 21.0.0 is still the latest stable release for Minecraft 26.3, but Fabric API
has a newer stable release, **0.166.2+26.3** (as of when this README was written). This
development environment's network access is restricted to an allow-list of domains and could not
reach Modrinth's or GitHub's release downloads directly, so 0.166.0+26.3 remains bundled. To
upgrade it yourself: download `fabric-api-0.166.2+26.3.jar` from Fabric API's version page on
Modrinth, replace `src/launcher/resources/triweave/fabric-api.jar`, and change
`fabric\_api\_version` in `bundled.properties` to `0.166.2+26.3`.

## Server edition (Fabric / Quilt / NeoForge + Bukkit-family detection)

The dedicated server uses the same `triweave-0.1.0.jar` as the client.

### Setup

1. Build a server with the official Fabric installer:
`java -jar fabric-installer.jar server -mcversion 26.3 -downloadMinecraft`
(this produces `fabric-server-launch.jar`)
2. In the same folder: `java -jar triweave-0.1.0.jar install-server`
(optional flags: `--dir <folder>` / `--launch <launch jar name>` / `--mem <e.g. 6G>`)
3. Start the server with the generated `start.bat` / `start.sh` (they just run
`fabric-server-launch.jar` with `-javaagent:triweave-x.y.z.jar` added).

Put mods in `mods/` and Bukkit-family plugins in `plugins/`. You still need to accept `eula.txt`
as with vanilla Fabric. The `/mod` command can only be used **from the server console, or by an
OP player** (enable/disable and reload require permission equivalent to `COMMANDS\_GAMEMASTER`,
i.e. OP level 2+; listing and info are open to everyone). The server never generates Mod Menu
stubs and never auto-installs `modmenu`.

### Bukkit / Spigot / Paper support (plugin.yml format; Paper's new `paper-plugin.yml` format and

Folia are not supported)

Using the same "evaluate → translate → run" pipeline as NeoForge, **plugins using the plugin.yml
format are actually executed**, within a deliberately narrow scope (because only a small slice of
the Bukkit API has been reimplemented):

* Plugin lifecycle (`onLoad` / `onEnable` / `onDisable`), `getLogger()`
* Config files (`getConfig()` / a basic YAML subset only — lists and multi-line strings aren't
supported)
* Event registration and dispatch via `Bukkit.getPluginManager().registerEvents(...)`
(`@EventHandler`)
* Commands (`getCommand(name).setExecutor(...)`)
* Sync/async tasks (`BukkitScheduler`; sync tasks run on the server tick)
* `ChatColor`

**Not supported**: any connection to real players, worlds, blocks or inventories
(`getOnlinePlayers()` is always empty), the permission system (`PermissionDefault` etc.),
persistence (`NamespacedKey` etc.), plugins that ship their real implementation inside a nested
jar via jar-in-jar/shadow packaging (rejected, since static analysis can't verify what's inside),
and Kotlin. In practice this targets "small, self-written plugins that only use config, logging,
simple events, and commands."

Acceptance uses the same mechanism as NeoForge (statically scanning the APIs used and comparing
against what the compatibility layer provides). Run
`java -jar triweave-0.1.0.jar check <plugin.jar>` to check ahead of time. All four real-world
plugins tested (Chunky, LuckPerms, voicechat, WorldEdit) are rejected — LuckPerms because its real
implementation is inside a nested jar, and the other three because they use many unsupported APIs.
A minimal hand-written test plugin (using only config, logging, events, commands, and the
scheduler) was verified end-to-end — accept, translate, start, tick, config save — on a simulated
Fabric environment; this has not been verified on real hardware.

## Installing (official launcher)

1. Install Fabric Loader for Minecraft 26.3 with the official Fabric installer.
2. Run:

```
   java -jar triweave-0.1.0.jar install
   ```

   (optional flags: `--mc <path to .minecraft>` / `--base <Fabric version id>`)

3. Launch the **Triweave 26.3** profile that was added to the launcher.
4. Mods go in `mods/` as usual (Fabric/Quilt/NeoForge can be mixed).

This procedure is for the client. See the "Server edition" section for dedicated servers.

## Unverified items

* The server edition (`install-server`, automatic preparation via `-javaagent`, server-side `/mod`
permission checks) has not been verified by actually running a dedicated server.
* Bukkit/Spigot/Paper support has been verified end-to-end on a simulated Fabric environment with a
hand-written minimal plugin (accept, translate, start, events, commands, scheduler, config), but
not on a real server. All four real-world plugins tested (Chunky, LuckPerms, voicechat, WorldEdit)
are rejected as out of scope.
* The new NeoForge events (`BlockEvent.BreakEvent`, `LivingDeathEvent`) were verified against the
real method signatures in the Fabric API 26.3 jar, but not on real hardware.
* The new Triweave APIs (`bus()`, `players()`, `broadcast()`, `onServerReady`) were verified against
a simulated server environment. However, the exact method names `players()` relies on
(`getUUID`, `getGameProfile`, `isOp`, etc.) could not be checked against the real 26.3 jar — these
are Mojang-mapping names that have been stable for a very long time, so they're likely fine, but
this is unverified.
* Since this environment cannot obtain the real Minecraft or Fabric Loader, **running on real
hardware is unverified** overall. What has been verified: compilation, jar classification, Quilt
translation (including nested jars), cache reuse, Fabric API auto-install, and the language
adapters (against stub Fabric API classes).
* Relies on `fabric.addMods` to hand mods to the Loader.
* Log lines start with `\[Triweave]` — check those first if something goes wrong.

## License

Triweave: MIT. Bundled Fabric API: Apache-2.0 (https://github.com/FabricMC/fabric).
Gson (Apache-2.0) is also bundled, for the installer.
