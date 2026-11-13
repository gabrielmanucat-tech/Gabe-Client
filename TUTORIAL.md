# Gabe Client: Build Tutorial (Fabric, Minecraft 26.2)

Gabe Client is a **cheat-free** Fabric client mod with a red-and-blue, web-themed menu.
Press **Right Shift** in a world to open it. Press it again (or Esc) to close.

**What it includes**

| Module | What it does |
|---|---|
| FPS Counter | Shows your frame rate |
| Coordinates | Shows your X / Y / Z |
| Facing Direction | Shows north / south / east / west |
| Keystrokes | Shows WASD + Space as you press them |
| Clock | Shows your computer's time |

**Why it is not a cheat.** Every module only *reads* information your game already has
and *draws* it on screen. Nothing sends packets, changes movement, touches other players,
reveals hidden blocks or entities, or automates clicks. When you add features later,
keep that rule: if it changes what the game does rather than what you see, leave it out.

> Note on the theme: the look uses red/blue colours and a spider-web pattern drawn in code.
> It does not use any Spider-Man character art or logos. If you ever share the mod publicly,
> keep it that way (no official logos or character images).

---

## Part 1: What you need installed

1. **JDK 25.** Minecraft 26.x runs on Java 25, so the mod must be built with it.
   Get it from Adoptium (Temurin 25) or Microsoft's OpenJDK 25.
   Check with `java -version` (it should say 25).
2. **IntelliJ IDEA Community** (free). This is the editor Fabric recommends.
3. **Minecraft Java Edition** with the **Fabric Loader** installed (for playing the finished mod).
   Install Fabric Loader for 26.2 from fabricmc.net, using the installer.
4. **Fabric API for 26.2** (a mod that goes in your `mods` folder). Download from Modrinth or CurseForge.

---

## Part 2: Create the project

There are two ways. **Way A is easier** because it gives you the Gradle wrapper files
(the `gradlew` scripts), which I cannot generate for you.

### Way A (recommended): generate a template, then drop in Gabe Client

1. Go to the **Fabric Template Mod Generator** (fabricmc.net/develop/template).
2. Fill it in:
   - Mod name: `Gabe Client`
   - Mod ID: `gabe-client`
   - Package name: `com.gabe.client`
   - Minecraft version: `26.2`
   - Leave Kotlin and data generation off.
3. Click Generate and unzip the result somewhere simple, for example `C:\Projects\gabe-client`.
   Avoid OneDrive, spaces, and accents in the path.
4. From this project, copy these over the generated ones (replace when asked):
   - `build.gradle`
   - `gradle.properties`
   - `settings.gradle`
   - the whole `src` folder
5. Delete the template's example code so it does not clash:
   - If a `src/client` folder exists, delete it.
   - Delete the template's example `ExampleMod` / `ExampleModClient` Java files and any
     `*.mixins.json` files inside `src/main`.
   - Make sure only `com/gabe/client/*.java` remains under `src/main/java`.

### Way B: use my files directly

Open this folder in IntelliJ. You will need the Gradle wrapper. If you have Gradle installed,
run `gradle wrapper --gradle-version 9.5.1` in the folder to create it. Otherwise use Way A.

---

## Part 3: Open it in IntelliJ

1. `File > Open`, pick the project folder, and trust it.
2. If IntelliJ asks about a Gradle build script, click **Import Gradle Project**.
3. Set the project JDK: `File > Project Structure > Project > SDK` = **25**.
   Also check `Settings > Build Tools > Gradle > Gradle JVM` = **25**.
4. Wait for the first sync. It downloads Minecraft and libraries, so it can take a few minutes.

---

## Part 4: The files, one at a time

### `gradle.properties`: the version numbers in one place

```
minecraft_version=26.2
loader_version=0.19.5
loom_version=1.17-SNAPSHOT
fabric_api_version=0.155.2+26.2
```

Two things changed from older tutorials you may have seen:
- There is **no `yarn_mappings`** any more. Minecraft 26.1+ ships unobfuscated, so you use the game's real names.
- Loom is version **1.17**, and Fabric API is `0.155.2+26.2` or newer. If Gradle cannot find that exact
  Fabric API version, look up the newest `+26.2` version on Modrinth and paste it in.

### `build.gradle`: how the project is built

Key lines and why they matter:
- `id 'net.fabricmc.fabric-loom'` is the **non-remapping** Loom plugin. It is the one for 26.1 and newer.
  (`fabric-loom-remap` is for old obfuscated versions.)
- `minecraft "com.mojang:minecraft:..."` tells Loom which Minecraft to set up.
- There is **no `mappings` line**.
- Dependencies use plain **`implementation`**, not `modImplementation`. Loom 1.17 removed the `mod*` ones.
- `options.release = 25` compiles for Java 25.

### `settings.gradle`: where Gradle finds the Fabric plugin

Adds the Fabric Maven repository so `net.fabricmc.fabric-loom` can be downloaded.

### `src/main/resources/fabric.mod.json`: the mod's ID card

- `"environment": "client"` means the mod only loads on the client, so it is never required on a server.
- `"entrypoints": { "client": [...] }` points at `GabeClientMod`.
- `"depends"` says it needs Fabric Loader, Minecraft 26.2, Java 25+, and Fabric API.

### `src/main/resources/assets/gabe-client/lang/en_us.json`: readable names

Gives the key category and key binding proper names in the Controls screen.
The category key format is `key.category.<modid>.<name>`.

### `Module.java`: the list of features

An enum with each feature's display name and whether it starts on. To add a feature,
add a line here (see Part 8).

### `GabeConfig.java`: saving your choices

Reads and writes `config/gabe-client.json` using Gson (already included with Minecraft).
`toggle(module)` flips a module and saves immediately. Missing entries fall back to defaults,
so adding modules later never breaks old config files.

### `GabeHud.java`: drawing the modules in-game

`extract(...)` runs every frame. It returns early if you are not in a world or if you pressed F1.
For each enabled module it draws a small dark box with a red accent bar and text.
The keystroke display only calls `isDown()` on your existing movement keys. It never presses anything.

Two names worth knowing in 26.x:
- The drawing object is **`GuiGraphicsExtractor`** (older versions called it `GuiGraphics`/`DrawContext`).
- Text uses `graphics.text(font, string, x, y, argbColor, shadow)`. Colours must be **ARGB**
  (for example `0xFFFFFFFF` is opaque white). If you pass plain RGB your text will be invisible.

### `GabeScreen.java`: the Spider-themed menu

- `init()` runs when the screen opens or the window resizes. It builds one button per module,
  plus a Close button, and pre-computes the web pattern.
- `extractRenderState(...)` runs every frame. The order matters:
  1. draw the dark backdrop and panel,
  2. call `super` (vanilla background and the buttons),
  3. draw the web, header, border, and footer on top.
- The web is built from straight spokes and sagging rings in two corners. Points are computed once in
  `init()` and just replayed each frame, which keeps it cheap. Pixels that would land on the panel are
  skipped, so the web never crosses your buttons.
- The border pulses between red and blue using real time (`Util.getMillis()`).
- `isPauseScreen()` returns false so the game keeps running behind the menu.

### `GabeClientMod.java`: wiring it all together

- Registers a key binding for **Right Shift** (`InputConstants.KEY_RSHIFT`) inside a "Gabe Client" category.
  Because it is a real key binding, you can rebind it in `Options > Controls > Key Binds`.
- A client tick listener opens the menu when the key is pressed and no other screen is open.
- A screen listener closes the menu when the same key is pressed while it is open.
- Attaches the HUD drawing right before the chat layer.

---

## Part 5: Run it in a test game

In IntelliJ, open the Gradle panel (right side) and run `Tasks > fabric > runClient`
(the name can be `runClient` directly under the project's tasks). Or from a terminal in the project folder:

```
./gradlew runClient        # Mac / Linux
gradlew.bat runClient      # Windows
```

A Minecraft window opens with your mod loaded. Create or join a world, then:
1. Press **Right Shift**. The menu should open.
2. Click **Keystrokes: OFF** to turn it on, then close the menu and press W, A, S, D.
3. Press Right Shift again to confirm it closes.

---

## Part 6: Build the jar and install it

1. Run `./gradlew build` (or `gradlew.bat build`).
2. Your mod is at `build/libs/gabe-client-1.0.0.jar`.
   Ignore the one ending in `-sources.jar`.
3. Put these two files in your `.minecraft/mods` folder:
   - `gabe-client-1.0.0.jar`
   - the Fabric API jar for 26.2
4. Launch the **Fabric 26.2** profile in the Minecraft launcher.
5. Join a world and press **Right Shift**.

---

## Part 7: Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `Unsupported class file major version` or Gradle fails at start | Wrong Java. Set both the project SDK and Gradle JVM to **25**. |
| `Plugin ... net.fabricmc.fabric-loom not found` | `settings.gradle` is missing the Fabric Maven repo, or Loom version is wrong. Use the files provided. |
| Cannot resolve `fabric-api:0.155.2+26.2` | Use the newest `+26.2` Fabric API version and update `fabric_api_version`. |
| Red squiggles on an import | Some package or method names shift between 26.x releases. Press Alt+Enter to auto-import, and check the current Fabric docs (docs.fabricmc.net, set to 26.2). |
| Text does not show | Colour is missing the alpha part. Use `0xFF` + RGB. |
| Menu opens but looks dim | Vanilla draws a dark background under the buttons. The theme is drawn around it. You can raise the alpha values in the panel gradient. |
| Game crashes at launch with "requires fabric-api" | Fabric API jar is missing from `mods`. |
| Nothing happens on Right Shift | Check `Options > Controls > Key Binds > Gabe Client`. Another mod may use the same key. |

**Important honesty note:** I wrote this against the current Fabric docs for 26.2
(Key Mappings, Custom Screens, Drawing to the GUI, and Rendering in the HUD pages), but I could not compile it in my
environment, so you are the first to build it. If the compiler complains about a single name, send me
the exact error text and I will fix it quickly.

---

## Part 8: Make it yours

**Change the colours.** In `GabeScreen.java` edit `RED`, `BLUE`, `NAVY`, `CRIMSON`, `WEB`.
Colours are `0xAARRGGBB`.

**Make the web denser or lighter.** In `addCornerWeb`, change `spokes` and `rings`.
More lines look richer but cost a little more per frame.

**Add a web to the other two corners.** In `buildWeb()` add:
```java
addCornerWeb(segments, this.width, 0, -1, 1);
addCornerWeb(segments, 0, this.height, 1, -1);
```

**Add a new module (example: show your health as a number).**
1. In `Module.java` add `HEALTH("Health", false),` before the semicolon.
2. In `GabeHud.extract(...)` add:
```java
if (GabeConfig.isEnabled(Module.HEALTH)) {
	y = line(graphics, mc, x, y, "HP", String.valueOf((int) player.getHealth()));
}
```
That is it. The menu builds its buttons from the enum automatically.

**Good cheat-free ideas:** armor durability display, ping display, day counter, a speedometer that
only reads your own speed, a custom crosshair colour, a screenshot shortcut.
**Avoid:** anything that fights, aims, flies, speeds you up, shows hidden things, or edits packets.

---

## Part 9: Play fair

Display-only HUD mods are generally accepted, but servers set their own rules.
Before using any client mod on a public server, read that server's rules.
