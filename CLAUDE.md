# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is actually in this repo

The only source file is `casa.html`: a self-contained, interactive 3D model (Three.js) of the AirCare two-story house mockup, including an optional layer showing where the sensors, actuators, ESP32 and wiring are installed.

**The README does not match the repo contents.** It documents Arduino/Wokwi firmware (`sketch.ino`, `diagram.json`, `wokwi.toml`, `firmware_esp32/aircare_esp32.ino`) that is not in this repository. Treat the README as the project's hardware reference (sensor list, GPIO pins, MQTT topics, system architecture), not as a description of the files here. Code, comments and UI text are all in Spanish; keep new code in Spanish too.

## Commands

```bash
npm install      # installs live-server (only dev dependency)
npm start        # or: npm run dev — serves the repo at http://localhost:5173/casa.html with live reload
```

There is no build step, linter or test suite. You can also open `casa.html` directly in a browser. Three.js **r128** is loaded from the cdnjs CDN as a global `THREE` (no modules or importmap), so a network connection is required. If the script fails to load, the page shows the `#aviso` message instead of calling `main()`.

## Architecture of `casa.html`

Everything lives inside a single `main()` function, split into sections marked with `// ====` banner comments. In order:

1. **Dimensions**: constants in centimeters (**1 unit = 1 cm**). `BW/BD/BT` is the base, `HX` is the house half-width, `HZB`/`HZF` are the back and front wall z-positions, `T` is the wall thickness, `H1`/`SLAB`/`H2` are the floor heights, and `yA = H1 + SLAB` is the second-floor level. Geometry is placed from these constants, so change them with care because other positions depend on them.
2. **Scene, camera, lights**: the page and the scene use a dark theme. The UI accent is midnight blue (`--azul: #191970` in `:root`), and the roof is brown (`mCafeTecho`). Because `renderer.outputEncoding` is sRGB, a plain hex material color renders lighter than written. When a color must match on screen (roof, fog, ground, burgundy cabinet, yellow armchair), wrap it in `lin(hex)`, which converts it to linear first.
3. **Procedural textures**: `tex(w, h, draw)` draws on a canvas and returns a `CanvasTexture`. No image assets are used.
4. **Materials**: `M(color, extra)` is a `MeshStandardMaterial` factory, and materials use `m*` names (`mBlanco`, `mMadera`, ...).
5. **Building helpers**: `caja` (box), `plano` (plane) and `cil` (cylinder) create meshes with shadows already set.
6. **Layer groups**: `gBase`, `gBajo`, `gAlto`, `gTecho`, `gMuebles`, `gTerraza` and `gSensores`. The `capas` map in the UI controls section links checkboxes (`c-techo`, `c-alto`, ...) to these groups. A new toggleable layer needs a group, a checkbox in the panel and an entry in `capas`. `gSensores` starts hidden.
7. **House geometry**: this is modeled on photos of the physical mockup.
   - The base has a rear "technical bay" (`bahía técnica`) for the electronics.
   - Ground floor: the kitchen is on the left and the bathroom on the right. They are split by the wall at `XDIV`, and the bathroom runs the full depth and is open on the right side.
   - The stairs rise toward -x and arrive at `XESC`.
   - The porch has a side wall with three openings and a flat white slatted pergola.
   - The back wall has an opening at floor level (`VANO`) and four holes (`PERFORACIONES`).
   - The right roof slope is built in pieces around the skylight opening (`TRAGA`).
   - `contorno()` and `lamina()` build flat shapes with holes and normalize their UVs so textures cover the whole piece.
   - Furniture was placed to clear the sensor mounts: the DHT22 PA post rests on top of the wardrobe, and the alert module sits above the bed. The dust sensor connector is why `XDIV` is -2.0. If you move furniture or walls, check them against the `registrar(...)` positions, which must not move.
8. **AirCare sensor layer**:
   - `mod*()` functions build each hardware module: `modMQ`, `modPolvo`, `modDHT22`, `modBH1750`, `modAlerta`, `modRelevadores`, `modESP32`, `modProtoboard` and `modVentilador`.
   - `montaje(modelo, texto, x, y, z, op)` places a module on a post and adds a label from `etiqueta()`.
   - `manojo(puntos, colores)` draws cable bundles. `ruta()` builds a standard path from a sensor, through the back wall, to a destination in the bay. `pasamuros` adds the wall pass-throughs.
   - Destinations are listed in the `L` object. Wire colors come from `CAB`.
   - The labels show the GPIO pins from the README's hardware table. If the pins change, update the labels to match.
   - Mounted modules are added through `registrar(id, montaje(...))`, which stores them in `montados` so the simulation can raycast clicks on them. Modules expose their animated parts through `userData`: fan `rotor`, alert `led`/`luz`, the relay board's per-channel `leds` and the ESP32's `ledAzul`.
9. **Orbit camera**: a custom controller, not `OrbitControls`. The `ctl` object stores `theta`, `phi`, `radio` and `objetivo`, and `actualizarCamara()` applies them. Presets are in `VISTAS` and are wired to `data-vista` buttons.
10. **Wokwi-style simulation** (the `#sim` panel on the right): `VARS` defines each sensor variable (range, example thresholds `acep`/`crit`, MQTT topic) and `ACTS` defines the actuators. Every `PERIODO_MS`, `ciclo()` imitates the firmware by logging the readings and optional MQTT publishes. `decidir()` then imitates the Node-RED decision tree and, in automatic mode, calls `comando()`. The ESP32 itself never classifies. `actualizarSim(dt, ahora)` runs in the render loop to spin the fans and blink the LEDs. The thresholds and scenarios (`ESCENARIOS`) are demo values, not calibrated ones.
11. **OBJ + MTL export**: `exportarOBJ()` walks the scene and exports only visible meshes, so hidden layers are left out. Objects with `userData.noExport = true` are skipped, including the ground plane, lights and label sprites. Set this flag on any new helper-only object. Materials are grouped by color (`mat_<hex>`), and the export downloads `casa_maqueta.obj` and `casa_maqueta.mtl`.
12. **Render loop**: resize handling and the optional auto-rotate (`girar`).

## Project context (from README)

AirCare IoT is a student project (Equipo 5) for monitoring indoor air quality in Colima. The ESP32 only takes measurements and follows commands. All classification logic runs in Node-RED, and an AI model provides the diagnosis. The ESP32 and Node-RED exchange data over MQTT: telemetry goes to `aircare/{pb,pa}/<var>` and commands to `aircare/cmd/{extractor,ventilacion,alerta}`. Because of the budget, the sensors give relative or qualitative readings only: no CO₂ ppm and no separate PM2.5/PM10.
