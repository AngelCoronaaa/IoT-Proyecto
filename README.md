# AirCare IoT — Código del Prototipo

**Equipo 5**
Corona Álvarez Ángel Valentín · Salazar Gutiérrez Jessica Paola · Luna Hernández Saúl · Magaña Rodríguez Valeria Michelle
Docente: Juan Antonio Guerrero Ibáñez

Monitoreo inteligente de la calidad del aire en espacios habitados de la
zona urbana de Colima, implementado como prototipo funcional sobre una
maqueta de vivienda de dos niveles a escala.

---

## 1. Qué contiene este repositorio

Este repositorio tiene **dos partes independientes de código**, que no se
ejecutan juntas y sirven para propósitos distintos:

| Carpeta | Qué es | Dónde corre |
|---|---|---|
| `/` (raíz) — `diagram.json`, `sketch.ino`, `wokwi.toml` | Simulación reducida con 2 sensores y 2 actuadores | Simulador Wokwi, dentro de VS Code |
| `/firmware_esp32/aircare_esp32.ino` | Firmware completo de producción, 5 sensores + 3 actuadores + MQTT | ESP32 físico, montado en la maqueta |

La simulación existe para probar la lógica de clasificación sin necesidad
de tener el hardware armado. El firmware de `/firmware_esp32` es el que
realmente se carga a la placa que va dentro de la maqueta.

```
aircare-proyecto/
├── README.md                      <- este archivo
├── diagram.json                   <- circuito de la simulación Wokwi
├── sketch.ino                     <- código de la simulación (2 sensores)
├── wokwi.toml                     <- configuración del simulador
└── firmware_esp32/
    └── aircare_esp32.ino          <- firmware real (5 sensores + MQTT)
```

---

## 2. Arquitectura del sistema (resumen)

El proyecto completo tiene cuatro capas. El código de este repositorio
cubre únicamente las dos primeras; Node-RED y la consulta al modelo de
IA se documentan y configuran aparte, fuera de este repositorio de
firmware.

1. **Percepción** — sensores y actuadores físicos sobre la maqueta.
2. **Comunicación** — el ESP32 publica lecturas y recibe comandos por
   MQTT. *(cubierta aquí)*
3. **Lógica** — Node-RED recibe la telemetría, aplica el árbol de
   decisión y publica los comandos de actuación. *(fuera de este repo)*
4. **Inteligencia** — un modelo de IA consultado desde Node-RED entrega
   diagnóstico, recomendación y predicción. *(fuera de este repo)*

El ESP32 **no clasifica nada**: solo mide y obedece. Toda la lógica de
decisión vive en Node-RED, para que una falla de red o de la IA nunca
impida que el sistema clasifique el ambiente ni accione el extractor.

---

## 3. Declaración de alcance de los sensores

El proyecto opera con una restricción presupuestal de **200 pesos por
sensor**. Por eso el sistema **no mide dióxido de carbono en ppm ni
material particulado diferenciado (PM2.5 / PM10)** — esos sensores de
concentración absoluta superan el presupuesto por un factor de 2 a 3.

En su lugar, el sistema mide **variables relativas**: densidad de polvo
relativa, presencia cualitativa de gases y compuestos volátiles, y humo.
Con ellas se construye un índice compuesto propio, calibrado empíricamente
sobre la maqueta. Los rangos de la NOM-172-SEMARNAT-2019 se usan como
referencia conceptual del esquema de tres estados (saludable / aceptable
/ crítico), no como umbrales de concentración verificables.

---

## 4. Sensores y actuadores

| # | Componente | Variable | Interfaz | Pin (firmware real) |
|---|---|---|---|---|
| 1 | Sharp GP2Y1010AU0F | Densidad de polvo (relativa) | Analógico + control de LED | AOUT: GPIO32 · LED: GPIO33 |
| 2 | MQ135 | Aire viciado / COV (cualitativo) | Analógico | GPIO34 |
| 3 | MQ2 | Humo y gases combustibles | Analógico | GPIO35 |
| 4 | DHT22 (planta baja) | Temperatura y humedad | Digital (1-Wire propietario) | GPIO4 |
| 5 | DHT22 (planta alta) | Temperatura y humedad | Digital (1-Wire propietario) | GPIO16 |
| 6 | BH1750 | Iluminación (lux) | I2C | SDA: GPIO21 · SCL: GPIO22 |
| 7 | Relevador — Extractor | Actuador | Digital | GPIO25 |
| 8 | Relevador — Ventilación cruzada | Actuador | Digital | GPIO26 |
| 9 | Relevador — Alerta roja (LED + buzzer) | Actuador | Digital | GPIO27 |

> Los pines son los definidos en `aircare_esp32.ino`. Si el cableado real
> de la maqueta usa otros GPIO, ajústalos ahí antes de compilar — ver
> sección 7.

**Nota de seguridad:** todos los actuadores operan en baja tensión (5V)
a través de relevadores. Ningún punto de la maqueta se conecta a la
tensión de la red doméstica.

---

## 5. Contrato de tópicos MQTT

El ESP32 publica telemetría y se suscribe a comandos. Node-RED hace lo
inverso: se suscribe a la telemetría y publica los comandos.

**Telemetría (ESP32 → Node-RED), payload JSON `{"ts": <ms>, "valor": <num>}`**

| Tópico | Contenido |
|---|---|
| `aircare/pb/pm` | Densidad de polvo, planta baja |
| `aircare/pb/voc` | Aire viciado (MQ135), planta baja |
| `aircare/pb/humo` | Humo / gas (MQ2), planta baja |
| `aircare/pb/temp` | Temperatura, planta baja |
| `aircare/pb/hum` | Humedad relativa, planta baja |
| `aircare/pa/temp` | Temperatura, planta alta |
| `aircare/pa/hum` | Humedad relativa, planta alta |
| `aircare/pa/lux` | Iluminación, planta alta |

**Comandos (Node-RED → ESP32), payload texto plano `"ON"` / `"OFF"`**

| Tópico | Acción |
|---|---|
| `aircare/cmd/extractor` | Enciende / apaga el extractor |
| `aircare/cmd/ventilacion` | Enciende / apaga la ventilación cruzada |
| `aircare/cmd/alerta` | Activa / desactiva la alerta roja |

---

## 6. Requisitos

- **Visual Studio Code**
- Extensión **Arduino** (Microsoft) — para compilar y cargar el sketch
- Extensión **Wokwi for VS Code** — solo para la simulación
- **Core ESP32** instalado vía Board Manager (`esp32 by Espressif Systems`)
- Librerías (Library Manager de la extensión Arduino):
  - `DHT sensor library` (Adafruit) + `Adafruit Unified Sensor`
  - `BH1750` (Christopher Laws) — solo para el firmware real
  - `PubSubClient` (Nick O'Leary) — solo para el firmware real
  - `ArduinoJson` (Benoit Blanchon) — solo para el firmware real
- Un broker MQTT accesible en la red (Mosquitto recomendado) — solo
  para el firmware real
- ESP32 DevKit físico y cable USB — solo para el firmware real

---

## 7. Cómo correr la simulación (Wokwi)

1. Instala las extensiones **Arduino** y **Wokwi for VS Code**.
2. Abre esta carpeta completa en VS Code (no un archivo suelto).
3. Abre `sketch.ino`. Con `Ctrl+Shift+P`:
   - `Arduino: Board Manager` → instala `esp32 by Espressif Systems`
     si no aparece "ESP32 Dev Module" como placa disponible.
   - `Arduino: Board Config` → selecciona **ESP32 Dev Module**.
   - `Arduino: Library Manager` → instala `DHT sensor library` (Adafruit).
   - `Arduino: Verify` → compila y genera los `.bin` en `build/`.
4. Abre `wokwi.toml` y confirma que la ruta `firmware` coincide con el
   nombre real del archivo generado en `build/` (a veces no es exacto).
5. Abre `diagram.json` y presiona el botón de **play** (▶). El play
   solo carga el binario ya compilado — si no compilaste en el paso 3,
   fallará.
6. Haz clic sobre el DHT22 y el MQ2 en el diagrama para simular valores.
   Observa el **Serial Monitor** de Wokwi para ver la clasificación y el
   LED / buzzer reaccionando en vivo.

Cada vez que edites `sketch.ino`, vuelve a correr `Arduino: Verify`
antes de dar play de nuevo — Wokwi no recompila automáticamente.

---

## 8. Cómo cargar el firmware real al ESP32

1. Abre `firmware_esp32/aircare_esp32.ino` en VS Code.
2. En la parte superior del archivo, edita:
   - `WIFI_SSID` y `WIFI_PASSWORD`
   - `MQTT_BROKER` (IP del equipo donde corre Mosquitto/Node-RED)
   - Los `PIN_*` si tu cableado real difiere de la tabla de la sección 4.
3. Instala las librerías `BH1750`, `PubSubClient` y `ArduinoJson`
   (además de `DHT sensor library`, ya usada en la simulación).
4. Conecta el ESP32 por USB. Con `Ctrl+Shift+P`:
   - `Arduino: Select Serial Port` → elige el puerto del ESP32.
   - `Arduino: Board Config` → **ESP32 Dev Module**.
   - `Arduino: Upload` → compila y carga directamente a la placa.
5. Abre el **Monitor Serial** (115200 baudios) para confirmar que:
   - se conecta al WiFi y muestra su IP,
   - se conecta al broker MQTT,
   - imprime una lectura de los 5 sensores cada 5 segundos.
6. Desde Node-RED (o con un cliente MQTT como MQTT Explorer), suscríbete
   a `aircare/#` para confirmar que la telemetría llega, y publica
   manualmente `"ON"` en `aircare/cmd/extractor` para confirmar que el
   relevador responde.

---

## 9. Calibración pendiente

Los umbrales usados en la simulación (`sketch.ino`) y las conversiones
del sensor de polvo (`aircare_esp32.ino`, función `leerPolvo()`) son
valores de arranque, **no** cifras normativas. Antes de la demostración
final, el equipo debe:

- Calibrar el GP2Y1010 comparando su lectura contra una fuente de polvo
  o humo conocida y a distancia fija.
- Ajustar los umbrales de MQ135 y MQ2 tras el periodo de estabilización
  del sensor (burn-in), que puede tomar varias horas la primera vez que
  se energiza.
- Definir en Node-RED los umbrales finales del árbol de decisión con
  base en estas lecturas reales, no con los valores de ejemplo del
  firmware.

---

## 10. Limitaciones conocidas

- El ESP32 no valida la integridad de las lecturas más allá de descartar
  `NaN` del DHT22; el filtrado y promedio móvil se implementan en
  Node-RED, no en este firmware.
- No hay reconexión con backoff exponencial; la reconexión WiFi/MQTT es
  simple y puede saturar el log si la red cae por tiempo prolongado.
- El firmware no persiste datos localmente: si se pierde la conexión
  MQTT, las lecturas de ese periodo se pierden (no hay buffer en SD ni
  en memoria flash).

---

## 11. Licencia y uso

Proyecto académico desarrollado para fines de evaluación. Uso educativo,
sin garantía de funcionamiento en entornos de producción o de monitoreo
regulatorio.
