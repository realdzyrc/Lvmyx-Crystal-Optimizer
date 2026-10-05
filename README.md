# Crystal Optimizer — Minecraft 1.16.5 (Fabric)

**Speed-of-light crystals.** A client-side Fabric mod that removes end crystals
client-side the *instant* you attack them, instead of waiting for the server
round-trip — then immediately re-aims your crosshair so the very next click
already targets the next crystal.

- Minecraft **1.16.5** · Fabric Loader **>= 0.14.25**
- **Made by Lvmyx** — shown as `Credits: Lvmyx` in the in-game Fabric mod list
- **No Fabric API required** — the jar works standalone in any mods folder
- Mod id: `lvmyxcrystaloptimizer`, version `1.0.0+1.16.5`
  (unique so it can coexist with the third-party `crystaloptimizer` mod)
- Output: `build/libs/lvmyxcrystaloptimizer-1.0.0+1.16.5.jar`
  (a copy also lives in the workspace root)

## What it does

1. **Instant removal** — when an `ATTACK` packet for an end crystal leaves the
   client, the crystal is removed immediately via `ClientLevel.removeEntity`
   (the same path the server's remove packet uses), so it vanishes without
   waiting for the server.
2. **Instant retarget** — right after removal the mod runs the vanilla
   crosshair pick (`GameRenderer.pick`), which vanilla only does once per tick.
   At high CPS this means every click in the same tick already aims at the next
   crystal instead of the dead one.
3. **Server stays authoritative** — the ATTACK packet is always sent. Suppressing it
   would stop the server from ever breaking the crystal, so no explosion would play
   and the client would desync, so this mod deliberately never cancels it.
4. **Accurate damage gate** — removal only happens when vanilla `Player.attack`
   would actually deal damage (base attack damage × cooldown scale, plus any
   enchantment bonus). Under Weakness with an empty hand the server keeps the
   crystal alive, so the client does too — no desync.

Toggle key: **H** (changeable in Controls → Lvmyx Crystal Optimizer).
Toggling shows an action-bar message and persists to
`config/lvmyxcrystaloptimizer.properties` (`enabled=true|false`). The name is
deliberately prefixed — another installed mod owns `crystaloptimizer.properties`,
and sharing one file would make the two mods overwrite each other's flag.

## Build

Requires **JDK 17**. Everything else is bootstrapped by the Gradle wrapper:

```bash
./gradlew build          # Linux / macOS / Git Bash
gradlew.bat build        # Windows Command Prompt
```

The finished jar lands in `build/libs/lvmyxcrystaloptimizer-1.0.0+1.16.5.jar`.

Stack: Gradle 8.10.2 + Fabric Loom 1.8.13 + JDK 17, compiling against
Minecraft 1.16.5 with official Mojang mappings, targeting Java 8 bytecode
(`options.release = 8`) so the jar runs on the Java 8 runtime of 1.16.5.

## Install

Drop `lvmyxcrystaloptimizer-1.0.0+1.16.5.jar` into your `.minecraft/mods` folder
with Fabric Loader 0.14.25 or newer. Client-side only — it never changes what
the server sees, only how fast your client reacts.

## How it works (internals)

| File | Role |
| --- | --- |
| `mixin/ConnectionMixin` | `@Inject` at HEAD of `Connection.send(Packet)`; on outgoing `ServerboundInteractPacket` with action `ATTACK` it hands the packet to the handler. It never cancels — `Connection.send` does not check the cancelled flag, so cancelling there throws and crashes the game. |
| `handler/CrystalAttackHandler` | Resolves the packet target with `packet.getTarget(level)`, checks the vanilla damage gate, calls `level.removeEntity(id)` and re-picks the crosshair. |
| `mixin/MinecraftMixin` | `@Inject` at HEAD of `Minecraft.tick()`; registers the keybind once (appending it to `Options.keyMappings` so it shows up in the Controls screen) and polls `consumeClick()` for the toggle. |
| `config/OptimizerConfig` | Persisted `enabled` flag in `config/lvmyxcrystaloptimizer.properties`. |

The mixin config declares `lvmyxcrystaloptimizer-refmap.json`; Loom generates it so
the injections resolve against intermediary names at runtime
(`Connection.send` → `class_2535.method_10743`, `Minecraft.tick` →
`class_310.method_1574`).

Note on `build.gradle`: `fabric.mod.json` placeholders are substituted as plain
text, not with Gradle's `expand`. `expand` runs the file through Groovy's
template engine, which would rewrite the `\n` escapes in the description into
real newlines and ship invalid JSON inside the jar.

## License

MIT
