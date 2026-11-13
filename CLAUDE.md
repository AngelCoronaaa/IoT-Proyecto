# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is actually in this repo

The only source file is `index.html`: a self-contained, interactive 3D model (Three.js) of the AirCare house mockup (a laser-cut 3 mm MDF Victorian dollhouse kit: ground floor plus an attic under a steep gable roof, open at the back), including an optional layer showing where the sensors, actuators, ESP32 and wiring are installed. `casa.html` is just a redirect to `./` so old links (iot.stardest.com/casa.html) keep working.

**The README does not match the repo contents.** It documents Arduino/Wokwi firmware (`sketch.ino`, `diagram.json`, `wokwi.toml`, `firmware_esp32/aircare_esp32.ino`) that is not in this repository. Treat the README as the project's hardware reference (sensor list, GPIO pins, MQTT topics, system architecture), not as a description of the files here. Code, comments and UI text are all in Spanish; keep new code in Spanish too.

## Commands

```bash
npm install      # installs live-server (only dev dependency)
npm start        # or: npm run dev — serves the repo at http://localhost:5173/ (index.html) with live reload
```

There is no build step, linter or test suite. You can also open `index.html` directly in a browser. Three.js **r128** is loaded from the cdnjs CDN as a global `THREE` (no modules or importmap), so a network connection is required. If the script fails to load, the page shows the `#aviso` message instead of calling `main()`.

## Architecture of `index.html`

Everything lives inside a single `main()` function, split into sections marked with `// ====` banner comments. In order:

1. **Dimensions**: constants in centimeters (**1 unit = 1 cm**), estimated from product photos, not measured. `BW/BD/BT` and `BZ0/BZ1` are the base, `HX` is the interior half-width, `HZB` is the open back edge, `HZF`/`ZF` are the inner/outer faces of the front wall, `T` is the MDF thickness, `H1`/`SLAB` are the ground-floor height and slab, `yA = H1 + SLAB` is the attic floor, `TOP` is where the roof starts on the walls, and `RISE`/`ANG`/`YR` define the main roof (ridge along x at `z = ZC`). `CGX`/`CGE`/`ANGC` define the front cross gable and `PZ`/`PR0`/`PR1`/`PL` the porch. Geometry is placed from these constants, so change them with care because other positions depend on them.
2. **Scene, camera, lights**: the page and the scene use a dark theme. The UI accent is midnight blue (`--azul: #191970` in `:root`). The house is raw MDF (`mMDF`) with dark laser-burnt edges (`mCanto`). Because `renderer.outputEncoding` is sRGB, a plain hex material color renders lighter than written. When a color must match on screen (MDF, fog, ground), wrap it in `lin(hex)`, which converts it to linear first. Two point lights inside the house light the installation seen through the open back.
3. **Procedural textures**: `tex(w, h, draw)` draws on a canvas and returns a `CanvasTexture`. No image assets are used. `texTeja` (fish-scale shingles) is mapped in centimeters: extruded pieces use their shape coordinates as UVs, so its `repeat` is `1 / tile size in cm`.
4. **Materials**: `M(color, extra)` is a `MeshStandardMaterial` factory, and materials use `m*` names (`mMDF`, `mCanto`, `mTeja`, ...). `MDF = [mMDF, mCanto]` is the material pair for extruded pieces (faces, edges). Textured materials set `userData.hexExport` so the OBJ export gets a real color.
5. **Building helpers**: `caja` (box) and `cil` (cylinder) create meshes with shadows already set. Laser-cut pieces are drawn as 2D outlines: `forma(points, holes)` with hole paths from `rectH`, `arcoH` (round-arch window) and `circH`, then `pieza(shape, thickness, mats)` extrudes along local +z. `aLado(obj, x0)` turns a piece drawn in (z, y) onto a side wall (it grows toward -x from `x0`). `faldonPieza(shape, d, n, origin)` places a roof panel drawn in (along the ridge, distance down the slope) from its down-slope direction `d` and outward normal `n`.
6. **Layer groups**: `gBase`, `gBajo` (walls, windows, door, trims), `gAlto` (attic floor), `gTecho`, `gPorche` and `gSensores`. The `capas` map in the UI controls section links checkboxes (`c-techo`, `c-alto`, `c-porche`, `c-sensores`) to these groups. A new toggleable layer needs a group, a checkbox in the panel and an entry in `capas`. `gSensores` starts hidden.
7. **House geometry**: this is modeled on product photos of the MDF kit, which has no furniture.
   - The house is open at the back on both levels. The back roof slope has a large opening (`HUECO`) to see into the attic.
   - Front wall (`muroFrente`): a double door (`PUERTA`), two ground-floor windows (`VENT_F`) and the cross-gable window (`VENT_H`). Each side wall is a gable with one round-arch window per level (`VL`).
   - The front roof slope is cut in two around the cross gable (`xMuesca`); the cross-gable panels end at the valley line.
   - Decorations: `cancel()` window frames with muntins, louvered shutters, `encaje()` gingerbread trim on the gables, ridge cresting and finials.
   - The porch has four posts, curved brackets, front railings, a shed roof and a small gable over the entrance with a pierced pediment.
   - The two right-side windows hold the fans, so they have no muntins (`sinBarras`). Their adapter plates are in `gSensores`.
8. **AirCare sensor layer**:
   - `mod*()` functions build each hardware module: `modMQ`, `modPolvo`, `modDHT22`, `modBH1750`, `modAlerta`, `modRelevadores`, `modESP32`, `modProtoboard` and `modVentilador`.
   - `montaje(modelo, texto, x, y, z, op)` places a module and adds a label from `etiqueta()`. With `op.pared = d` it hangs on an L bracket from a wall `d` cm away in local -z, and `op.rotY` turns it toward that wall (`IZQ`, `DER`, `FRENTE`). Without it, the module sits on a post of length `op.poste`.
   - Layout: the electronics (protoboard + ESP32, and the relay board turned so its terminals face the right wall) sit on the back strip of the ground floor. Ground-floor sensors are on the left wall (MQ135, MQ2 high, DHT22 PB) and above the left front window (GP2Y1010). Attic: DHT22 PA on the left gable, BH1750 under the cross-gable window, alert on the right gable. The extractor and the cross-ventilation fan sit in the right-side windows.
   - `manojo(puntos, colores)` draws cable bundles along explicit point lists in `cables`. They run down the walls into `canaleta()` ducts: one along the floor on each side and one vertical duct in each back corner that goes up to the attic. Sensors end on the protoboard; actuators end at the relay terminals (`RELE`). Wire colors come from `CAB`.
   - The labels show the GPIO pins from the README's hardware table. If the pins change, update the labels to match.
   - Mounted modules are added through `registrar(id, montaje(...))`, which stores them in `montados` so the simulation can raycast clicks on them. Modules expose their animated parts through `userData`: fan `rotor`, alert `led`/`luz`, the relay board's per-channel `leds` and the ESP32's `ledAzul`.
9. **Orbit camera**: a custom controller, not `OrbitControls`. The `ctl` object stores `theta`, `phi`, `radio` and `objetivo`, and `actualizarCamara()` applies them. Presets are in `VISTAS` (each with a target `y` and `z`) and are wired to `data-vista` buttons. The `atras` preset looks through the open back at the installation.
10. **Wokwi-style simulation** (the `#sim` panel on the right): `VARS` defines each sensor variable (range, example thresholds `acep`/`crit`, MQTT topic) and `ACTS` defines the actuators. Every `PERIODO_MS`, `ciclo()` imitates the firmware by logging the readings and optional MQTT publishes. `decidir()` then imitates the Node-RED decision tree and, in automatic mode, calls `comando()`. The ESP32 itself never classifies. `actualizarSim(dt, ahora)` runs in the render loop to spin the fans and blink the LEDs. The thresholds and scenarios (`ESCENARIOS`) are demo values, not calibrated ones.
11. **OBJ + MTL export**: `exportarOBJ()` walks the scene and exports only visible meshes, so hidden layers are left out. Objects with `userData.noExport = true` are skipped, including the ground plane, lights and label sprites. Set this flag on any new helper-only object. Materials are grouped by color (`mat_<hex>`), and the export downloads `casa_maqueta.obj` and `casa_maqueta.mtl`.
12. **Render loop**: resize handling and the optional auto-rotate (`girar`).

## Project context (from README)

AirCare IoT is a student project (Equipo 5) for monitoring indoor air quality in Colima. The ESP32 only takes measurements and follows commands. All classification logic runs in Node-RED, and an AI model provides the diagnosis. The ESP32 and Node-RED exchange data over MQTT: telemetry goes to `aircare/{pb,pa}/<var>` and commands to `aircare/cmd/{extractor,ventilacion,alerta}`. Because of the budget, the sensors give relative or qualitative readings only: no CO₂ ppm and no separate PM2.5/PM10.
