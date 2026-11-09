# Proyecto IoT: Mapa de Vibraciones para diagnóstico de tren delantero (Cirfren)

> **Archivo de contexto del proyecto (v2).** Contiene todo lo definido hasta ahora: el problema, los alcances, las decisiones técnicas alineadas con lo dictado en la materia, la metodología de trabajo, los hitos y el estado actual.
> Claude: leé este archivo completo antes de responder, en cualquier chat o sesión del proyecto. Respetá las decisiones, las convenciones del cursado (sección 3) y el contrato de datos (sección 6).

---

## 0. Cómo se usa este archivo

- **Fuente de verdad:** este archivo vive en el repositorio del proyecto, en la raíz, con el nombre **`CLAUDE.md`**. Así Claude Code (la extensión de VS Code) lo lee automáticamente en cada sesión.
- **Copia en el proyecto de Claude.ai:** se sube como conocimiento del proyecto para los chats de planificación, pruebas e informe. Cuando cambia en el repositorio, se vuelve a subir.
- La sección **"11. Estado actual"** se actualiza cada vez que se cumple un hito o se toma una decisión. En Claude Code, Claude puede editarla directamente cuando Martín lo pide. En Claude.ai, Claude propone el texto y Martín lo actualiza.
- Si algo de lo que se habla contradice este archivo, Claude lo avisa y propone el texto actualizado.

---

## 1. Quién soy y para qué es el proyecto

- **Autor:** Martín (Juan Martín Perret Zuza), estudiante avanzado de Ingeniería en Computación (FIO-UNaM).
- **Materia:** *Internet de las Cosas, Sensores y Redes* (IC511). Cursada en 2026.
- **Trabajo:** individual. Recursos: mi computadora, una Raspberry Pi Pico 2W, sensores accesibles y presupuesto de estudiante.
- **Requisitos de la materia para aprobar:**
  - Haber regularizado la materia.
  - Hacer un **proyecto integrador individual y original**, desarrollado específicamente para esta materia.
  - Que la cátedra **apruebe el tema antes**.
  - Que cualquier uso posterior del proyecto se referencie como trabajo hecho en el marco de la materia "Internet de las Cosas, Sensores y Redes".
  - **Presentarlo** en al menos una jornada de divulgación científica o tecnológica (presentación personal e individual). Se evalúan las competencias CG7 (comunicación efectiva) y CG8 (ética, responsabilidad y compromiso social).
  - La nota depende del tipo de evento: exposición local de prototipos = 6; ponencia local = 7; evento nacional = 8; evento internacional = 9. La presentación presencial suma +1, salvo en eventos locales de pregrado o no profesionales. Se admiten presentaciones virtuales.
- **Evento previsto:** una exposición local de prototipos, con una versión funcional lista para el **jueves 19/11/2026**. Los hitos no tienen fechas porque el tiempo que le dedico varía, pero el camino crítico (sección 8) apunta a llegar a ese evento.
- **La cátedra también pide** (según la planificación): demostración en vivo, un informe con el problema, la estrategia, los resultados y el código, y el código en un repositorio Git.

---

## 2. El problema real: taller Cirfren

- El taller familiar **Cirfren** (https://cirfren.com.ar/) ofrece diagnóstico, revisión pre-VTV (con frenómetro, banco de suspensión y diagnóstico electrónico), frenos, embragues, tren delantero, suspensión y dirección (bujes, rótulas, extremos, bieletas, cazoletas), y alineación y balanceo.
- **La idea de Gustavo** (mi hermano, que trabaja todos los días en el taller): un dispositivo que complemente el diagnóstico actual y ayude a encontrar **de dónde viene un ruido o una vibración** (precap, extremo, rótula, cazoleta, parte alta o baja, izquierda o derecha). Lo imagina mostrando en el celular, en vivo, un **croquis del auto** con el tren delantero y el trasero, y las zonas coloreadas de verde a rojo según la intensidad de vibración. También menciona medir temperatura y obtener un "prediagnóstico".

### Qué es viable y qué no
| Idea | Veredicto | Cómo se aborda |
|---|---|---|
| Croquis en el celular con zonas coloreadas en vivo | ✅ Es el núcleo del proyecto | **Varios sensores, uno por zona, y comparación entre zonas**: la zona que vibra más que su par se pinta de rojo. |
| Medir temperatura (cáliper trabado, rodamiento) | ✅ Como extensión | Sensor IR MLX90614 para comparar ruedas. Cuidado: el metal pulido tiene baja emisividad, así que se mide sobre el cáliper o la maza y se compara entre ruedas. |
| Un solo aparato "en el medio" que diga de qué pieza viene el ruido | ❌ Fuera de alcance | Localizar una fuente con un solo sensor no es posible; con micrófonos sincronizados sería un proyecto de otra escala. |
| Prediagnóstico automático de la pieza | ⚠️ Solo por reglas | El MVP usa reglas simples ("zona X: 3× más golpes que su par → revisar..."). Un clasificador con aprendizaje automático queda como trabajo futuro: requiere un dataset etiquetado, que este prototipo empieza a generar. |

**Mensaje central para la presentación:** el sistema indica **dónde** y **qué tipo** de vibración hay (continua o con golpes, grave o aguda). **Qué pieza es** lo sigue decidiendo el mecánico. Es una herramienta que acorta la búsqueda, igual que el banco de suspensión da valores y no un veredicto.

**Antecedente comercial:** el ChassisEAR (JS Products/Steelman) usa varios micrófonos con pinzas sobre distintos componentes y un selector de canales para escuchar con auriculares. Cuesta varios cientos de dólares. Este proyecto **agrega medición objetiva, visualización, registro de datos y comparación automática**. Las críticas de los usuarios a ese equipo son útiles: el ruido ambiente tapa la señal y los micrófonos no tienen la misma sensibilidad. Por eso este proyecto usa acelerómetros con fijación rígida y calibración entre sensores.

---

## 3. Base del cursado: herramientas y convenciones usadas en clase

> **Regla:** el proyecto reutiliza lo visto en clase siempre que sirva. Si se usa algo nuevo, se justifica en `docs/decisiones.md`.

### Repositorios de referencia
- Profesor (con todo el contenido en **ramas**): `https://github.com/GermanXander/rpipico` y `https://github.com/GermanXander/docker_iot`.
- Mis forks: `https://github.com/Martincitok/rpipico` y `https://github.com/Martincitok/docker_iot` (solo tienen `main`, sin las ramas de las clases).
- Ramas útiles para este proyecto:
  - **rpipico:** `mqttb` y `BEb` (mqtt_as + JSON), `mqtt_events` (API de eventos de mqtt_as), `async` y `multi-task` (asyncio).
  - **docker_iot:** `BEe` (compose completo: MariaDB, phpMyAdmin, clienteMqtt, Mosquitto con TLS, Grafana, Portainer, swag), `CRUDsvg` (Flask + gunicorn + login + Bootstrap + matplotlib SVG), `FEa`–`FEc` (Grafana), `TBa`–`TBd` (bot de Telegram), `NRa`–`NRd` (Node-RED), `WWWa`–`WWWd` (dominio y TLS), `WG` (WireGuard).

### Lo que se usó, por unidad
| Unidad | Lo usado en clase |
|---|---|
| IDE | VSCodium/VS Code con la extensión *Raspberry Pi Pico*; Remote-SSH para trabajar en la VPS. |
| Pico + MicroPython | `main.py` (sin `boot.py`), librerías en `lib/`, `asyncio`, WLAN, DHT22. **MQTT con `mqtt_as` (Peter Hinch, asincrónico, TLS)**, configurado en `lib/mqtt_local.py`, y credenciales en **`settings.py`** (fuera del repositorio, en `.gitignore`). |
| VPS | Debian por alumno sobre Proxmox (IP interna de la facultad, 10.90.10.x), usuario `iot`, SSH, entornos virtuales, `requirements.txt` y `.env`. |
| Docker | Imagen propia con `FROM python:3.11-slim`, `TZ=America/Argentina/Buenos_Aires`, `compose.yaml`, variables de entorno desde `.env`, `docker logs`. |
| Backend | **MariaDB** + phpMyAdmin; **clienteMqtt** en Python con `aiomqtt` + `aiomysql` (se suscribe, decodifica el JSON e inserta en MariaDB, usando el último nivel del tópico como id del nodo); **Mosquitto** en contenedor (`mosquitto.conf`, `passwd`). |
| Frontend | **Grafana** sobre MariaDB. |
| WWW | Dominio con **DuckDNS**; **swag** (nginx + Let's Encrypt con validación por DNS); servicios publicados en subcarpetas; un puerto mayor a 10000 en la facultad (no se pueden usar 80 ni 443); **Mosquitto con TLS en el 8883** usando los certificados de swag. |
| Flask (CRUD) | Flask + **gunicorn** en Docker, `flask_mysqldb`, plantillas Jinja, ProxyFix detrás de nginx, login con hash scrypt, Bootstrap, gráficos SVG generados con matplotlib. |
| Bot de Telegram | `python-telegram-bot` (handlers asincrónicos), consultas con `aiomysql` y gráficos con matplotlib enviados como foto. |
| Node-RED | Contenedor, nodos MQTT, **InfluxDB**, bot de Telegram y Dashboard de FlowFuse. |
| Otros | WireGuard y LoRa (vistos en el cronograma). |

### Advertencia detectada en el código de clase (a verificar en H4)
En `lib/mqtt_local.py`, el LED se elige con `'Pico W' in implementation._machine`. En una **Pico 2W**, ese texto dice "Pico 2 W", así que la condición falla y se configura **GPIO25** como LED. En las placas con WiFi, GPIO25 está conectado al chip inalámbrico, y esto podría interferir con el WiFi. Si aparecen problemas de conexión, la solución es usar `Pin("LED", ...)` o no definir `blue_led`.

---

## 4. Hardware

### Disponible hoy
- 1 × Raspberry Pi Pico 2W (RP2350, WiFi), **con MicroPython ya instalado**.
- 1 × **MPU6050** (módulo GY-521): **es el sensor del MVP**.
- 1 × fuente o powerbank.
- El ADXL345 **no se consiguió** localmente (2026-11-08). Si aparece, se evalúa migrar los dos nodos.

### Previsto
- Una segunda Pico 2W y un segundo **MPU6050 (el mismo modelo, nunca mezclar modelos entre zonas)**.
- Caja estanca, imanes de neodimio, placa perforada, pines, cables mallados y precintos.

### Sensor: MPU6050 (actual) vs ADXL345 (alternativa)
| | MPU6050 (en uso) | ADXL345 (alternativa) |
|---|---|---|
| Mide | Aceleración + giroscopio + temperatura | Solo aceleración |
| Ancho de banda útil del acelerómetro | **~184–260 Hz** (ODR 1 kHz) | Hasta ~1600 Hz (ODR 3200 Hz) |
| Resolución a ±16 g | 2048 LSB/g (~0,5 mg) | ~3,9 mg/LSB |
| Interfaz | Solo I2C (0x68 / 0x69) | I2C (0x53 / 0x1D) y SPI |
| FIFO | **1024 bytes ≈ 170 muestras (170 ms a 1 kHz)** | 32 muestras (40 ms a 800 Hz) |
| Orden de bytes | **Big-endian** | Little-endian |
| Observaciones | Discontinuado y con muchos clones (verificar WHO_AM_I) | Vigente |

**Implicancia para el proyecto (límite conocido):** con el MPU6050, el sistema detecta bien los **golpes** (bujes, rótulas, bieletas, cazoletas con juego), la vibración de suspensión, el desbalanceo y el motor, y la **comparación entre zonas** no se ve afectada. **No** detecta vibraciones agudas por encima de unos 260 Hz (zumbido de rodamientos, ruidos metálicos finos). En la presentación se plantea como un límite conocido, con la migración a un sensor de mayor ancho de banda como mejora directa.

**Regla:** todos los nodos usan el mismo modelo de sensor. Si se migra al ADXL345, se migran todos.

### Conexión MPU6050 (GY-521) → Pico 2W (I2C, dirección 0x68)
| GY-521 | Pico 2W |
|---|---|
| VCC | 3V3(OUT), pin 36 (el módulo trae regulador) |
| GND | GND, pin 38 |
| SDA | GP4, pin 6 |
| SCL | GP5, pin 7 |
| AD0 | GND → 0x68 (a 3V3 → 0x69) |
| INT | Sin conectar en el MVP (se puede usar para FIFO/overflow) |
| XDA, XCL | Sin conectar |

### Registros clave del MPU6050
| Registro | Dirección | Valor | Significado |
|---|---|---|---|
| WHO_AM_I | 0x75 | lectura = 0x68 | Original. Otros valores (0x70, 0x72, 0x98…) indican un clon o un MPU6500/9250: anotarlo. |
| PWR_MGMT_1 | 0x6B | 0x01 | Despierta el sensor; reloj del PLL del giroscopio X |
| CONFIG | 0x1A | 0x01 | DLPF de 184 Hz (base de muestreo de 1 kHz). 0x00 = 260 Hz, pero la base pasa a 8 kHz. |
| SMPLRT_DIV | 0x19 | 0x00 | fs = 1 kHz / (1 + div) = 1000 Hz |
| ACCEL_CONFIG | 0x1C | 0x18 | ±16 g → **2048 LSB/g** |
| ACCEL_XOUT_H..ZOUT_L | 0x3B–0x40 | — | 3 × int16 **big-endian** (`struct.unpack('>hhh', ...)`) |
| USER_CTRL | 0x6A | 0x04 y luego 0x40 | Reinicia el FIFO; habilita el FIFO |
| FIFO_EN | 0x23 | 0x08 | Solo el acelerómetro al FIFO (6 bytes por muestra) |
| INT_STATUS | 0x3A | bit 4 | FIFO_OFLOW (desborde) |
| FIFO_COUNTH/L | 0x72–0x73 | — | Bytes en el FIFO (big-endian) |
| FIFO_R_W | 0x74 | — | Lectura del FIFO (leer múltiplos de 6 bytes) |
| TEMP_OUT | 0x41–0x42 | — | Temperatura del chip (extra) |

### ADXL345 (referencia por si se migra)
Conexión I2C: VCC→3V3, GND, SDA→GP4, SCL→GP5, CS→3V3, SDO→GND (0x53). Registros: DEVID 0x00 = 0xE5; BW_RATE 0x2C = 0x0D (800 Hz); POWER_CTL 0x2D = 0x08; DATA_FORMAT 0x31 = 0x0B (full resolution, ±16 g, 0,0039 g/LSB); datos 0x32–0x37 little-endian; FIFO_CTL 0x38 = 0x80 | watermark; FIFO_STATUS 0x39.

### Consideraciones del sensor
- **Driver con una interfaz común:** `lib/sensores/mpu6050.py` y `lib/sensores/adxl345.py` exponen los mismos métodos (`iniciar(fs, rango_g)`, `leer_fifo() → lista de (x, y, z) en g`, `desbordes()`). En `settings.py`, `SENSOR = "mpu6050"` elige el driver. El resto del firmware no depende del sensor.
- **Rango de ±16 g**, para no saturar con pozos o lomos de burro.
- **Leer el FIFO por bloques** de múltiplos de 6 bytes. A 1 kHz, el FIFO del MPU6050 se llena en unos 170 ms. Si se desborda, hay que reiniciarlo (USER_CTRL) para no quedar desalineado.
- **Convivencia con asyncio y mqtt_as:** la tarea de adquisición tiene que vaciar el FIFO antes de que se llene, aunque el cliente MQTT esté ocupado (por ejemplo, durante el handshake TLS). Se verifica en H2 y H4 contando los desbordes.
- **Quitar la gravedad** (restar el promedio o filtrar con un pasa-altos) antes de calcular la vibración.
- **Fijación rígida** (imán, tornillo o precinto bien ajustado). La cinta doble faz amortigua y genera resonancias propias.
- **Ubicaciones simétricas** y con la **misma orientación** en cada lado.
- **Calibración:** poner los dos sensores en el mismo punto y después intercambiarlos de lado. Los clones pueden diferir más entre sí, así que la calibración es todavía más importante.
- Temperatura de operación: hasta 85 °C. Mantenerlo lejos del escape.
- I2C no está pensado para cables largos: usar menos de 1 m y 100 kHz si hay ruido eléctrico.

---

## 5. Arquitectura (alineada con el cursado)

```
[MPU6050] --I2C--> [Pico 2W · MicroPython · asyncio · mqtt_as]
   (un nodo por zona, NODE_ID en settings.py)
                         │  MQTT (1883 en red local / 8883 con TLS hacia la VPS)
                         ▼
                 [Mosquitto (Docker)]
                         │
                         ▼
       [clienteMqtt: aiomqtt + influxdb-client] ──► [InfluxDB 2.x]  (UI propia en :8086)
                                                         │
                       ┌─────────────────────────────────┼─────────────────────────┐
                       ▼                                 ▼                         ▼
     [Flask + gunicorn: web "en vivo"]            [Grafana: historial]   [Bot Telegram (opcional)]
      croquis SVG del auto; JS consulta             gráficos por zona    informe al cerrar la sesión
      /api/ultimo cada ~0,5–1 s (JSON)              y por sesión
```

### Decisiones de diseño
- **Base de datos: InfluxDB 2.x** (base de series temporales, vista en la unidad de Node-RED). Motivos: el dato es una serie temporal por naturaleza (tiempo + zona + métricas); las consultas por ventana y el "último valor por zona" son nativas; se integra naturalmente con Grafana; y escala si después se guardan datos crudos para la FFT. **La imagen se fija como `influxdb:2`** (no `latest`) para mantener la interfaz, la organización, el bucket, el token y Flux, como en clase. MariaDB queda descartada para este proyecto (ver `decisiones.md`).
- **Ingestor:** se adapta el `clienteMqtt` de clase, con aiomqtt para la suscripción y `influxdb-client` para la escritura, en lugar de aiomysql. El cliente es único y persistente. Plan B si hay problemas: un flujo de Node-RED (`mqtt in` → `function` → `influxdb out`), como en clase.
- **Hora del punto:** se usa el `ts` del nodo. Si difiere más de ±5 s de la hora de recepción (porque falló el NTP), se usa la hora de recepción y se registra un aviso.
- **Vista en vivo: Flask + gunicorn** (como el CRUD) sirviendo una página con el **croquis SVG**. El JavaScript del navegador consulta un endpoint JSON (`/api/ultimo`) cada 0,5–1 s y colorea las zonas. Así no hace falta MQTT por WebSockets (que no se vio en clase), y la latencia es aceptable para el uso.
- **Historial: Grafana** con InfluxDB como fuente de datos (Flux), con un panel por zona y la comparación entre pares. Reutiliza lo visto en Frontend.
- **Firmware: asyncio + `mqtt_as`**, como en clase. `mqtt_as` ya maneja la conexión y la reconexión WiFi/MQTT y el Last Will. Credenciales y `NODE_ID` en `settings.py`.
- **Comparar entre zonas, no mirar valores absolutos.** El ruido de fondo (motor, calzada) llega a todos los sensores y la diferencia entre ellos es la señal útil. El acelerómetro mide vibración estructural, no sonido en el aire.
- **El nodo procesa y el servidor analiza.** La Pico calcula indicadores por ventana (RMS, pico, golpes). El análisis pesado (FFT, espectros) se hace en el servidor o en `analisis/` con numpy.
- **Firmware genérico.** Un nodo nuevo es copiar el código y cambiar el `NODE_ID` en `settings.py`.
- **Nodo simulado** (script de Python con aiomqtt) que publica con el mismo contrato, para desarrollar con varias zonas mientras haya un solo nodo real.
- **Medición por turnos** mientras haya un solo nodo: misma prueba repetible (ideal en el banco de suspensión), primero un lado y después el otro.
- **Dos perfiles de despliegue:**
  - **Local / expo:** `compose.yaml` en la notebook con Mosquitto (1883, sin TLS, con usuario y contraseña), InfluxDB, clienteMqtt, Flask y Grafana. Sin swag. Todo en una red WiFi propia (hotspot).
  - **VPS / remoto:** el mismo stack más swag, DuckDNS y Mosquitto con TLS en el 8883, como en la rama BEe y en WWW.

### Indicadores por ventana (≈ 500 ms)
- **RMS** por eje y total (en g, sin gravedad): nivel de vibración.
- **Pico** (en g): máximo absoluto de la ventana.
- **Golpes:** cantidad de eventos que superan k × RMS de referencia (factor de cresta). Distingue impactos (bujes o rótulas con juego) de la vibración continua.
- **Relación entre pares** (der/izq), calculada en Flask: es la base del color.

### Qué ve el operario (tres niveles)
1. **Croquis del auto** con zonas coloreadas según la vibración relativa a su par (verde, amarillo, naranja, rojo) y el estado online/offline de cada nodo.
2. **Indicadores por zona:** RMS, golpes por minuto, relación con el par (por ejemplo, "2,7×") y un gráfico simple en el tiempo.
3. **Mensaje por reglas**, por ejemplo: "Zona delantera derecha: golpes frecuentes, 3× más que la izquierda → revisar cazoleta, bieleta o amortiguador derecho".
Además, el historial y el análisis por sesión en Grafana.

### Conectividad
| Escenario | Solución |
|---|---|
| Taller / expo | Red local propia: hotspot del celular o de la notebook; la notebook corre el stack local. **En la expo no depender del WiFi del lugar.** |
| Prueba en ruta | **Opción A:** hotspot del celular con datos móviles y broker en la VPS, accesible por el dominio y el puerto TLS (hay que verificar que la VPS sea alcanzable desde fuera de la facultad; si no, se usa WireGuard o la opción B). **Opción B:** notebook en el auto con el stack local y el hotspot. |
| Respaldo (opcional) | La Pico guarda resúmenes en flash y los sube al volver a tener WiFi. |

Otras notas: la carrocería atenúa el WiFi, así que hay que verificar el RSSI desde el vano motor. Y **el que mira el celular en la prueba en ruta es el acompañante, nunca el conductor.**

---

## 6. Contrato de datos (interfaz entre módulos)

> Cualquier cambio a este contrato se decide en el chat de Coordinación y se actualiza acá.

### Identificadores de zona (`NODE_ID`)
`del_izq`, `del_der`, `tra_izq`, `tra_der` (se pueden agregar otras, como `del_izq_inf` para la parte baja).
**Pares para comparar:** `del_izq ↔ del_der`, `tra_izq ↔ tra_der`.

### Tópicos MQTT
| Tópico | Contenido | QoS / retain |
|---|---|---|
| `cirfren/vib/<node_id>/metricas` | Indicadores por ventana (JSON) | QoS 0 |
| `cirfren/vib/<node_id>/estado` | `"online"` / `"offline"` (Last Will, `config['will']` en mqtt_as) | QoS 1, retain |
| `cirfren/sesion` | Inicio y fin de una prueba (patente, tipo de prueba). Lo publica la web de Flask. | QoS 1, retain |

El clienteMqtt se suscribe a `cirfren/#` y toma el `node_id` del tópico, igual que en clase.

### Payload de `metricas`
```json
{
  "node": "del_der",
  "ts": 1731000000123,
  "seq": 152,
  "fs": 1000,
  "sensor": "mpu6050",
  "win_ms": 500,
  "rms": {"x": 0.012, "y": 0.009, "z": 0.031, "total": 0.035},
  "peak_g": 0.41,
  "hits": 3,
  "ovf": 0,
  "rssi": -67
}
```
- `sensor`: modelo usado por el nodo (para no mezclar modelos al comparar). `ts`: milisegundos Unix (reloj sincronizado por NTP con `ntptime`). `seq`: contador para detectar ventanas perdidas. `ovf`: desbordes del FIFO en la ventana.
- Unidades: g, sin gravedad.

### Payload de `sesion`
```json
{"accion": "inicio", "sesion_id": "2026-11-15_AB123CD_banco", "patente": "AB123CD", "prueba": "banco_suspension", "ts": 1731000000000}
```

### Esquema en InfluxDB (borrador)
- **Organización:** `cirfren` · **Bucket:** `vibraciones` (retención infinita en el MVP). Si después se guardan datos crudos: bucket `crudo` con retención de 30 días.
- **Measurement `metricas`:**
  - *Tags:* `node` (`del_der`…), `eje` (`del` / `tra`), `lado` (`izq` / `der`), `sesion` (`sesion_id` activa o `ninguna`), `sensor` (`mpu6050` / `adxl345`).
  - *Fields:* `rms_x`, `rms_y`, `rms_z`, `rms_total`, `peak_g`, `hits`, `seq`, `ovf`, `rssi`, `fs`, `win_ms`.
  - *Hora:* `ts` del nodo (precisión en ms), con la regla de respaldo de la sección 5.
- **Measurement `sesiones`:** *tags* `sesion`, `patente`, `prueba`; *field* `accion` (`inicio` / `fin`). El ingestor guarda la sesión activa (del mensaje retenido en `cirfren/sesion`) y etiqueta cada punto de `metricas` con ella.
- **Consulta "último valor por zona"** (la usa `/api/ultimo`):
  ```flux
  from(bucket: "vibraciones")
    |> range(start: -10s)
    |> filter(fn: (r) => r._measurement == "metricas")
    |> last()
    |> pivot(rowKey: ["node"], columnKey: ["_field"], valueColumn: "_value")
  ```

---

## 7. Estructura del repositorio

Es un **repositorio nuevo y propio** del proyecto (el trabajo tiene que ser original). Sigue las convenciones de los repositorios de clase y cita de dónde se toma código base (mqtt_as, con licencia MIT de Peter Hinch, y los ejemplos de la cátedra).

```
cirfren-vibraciones/
├── CLAUDE.md                 # este archivo
├── README.md                 # incluye la mención a la materia
├── firmware/                 # se copia a la Pico (estructura como rpipico)
│   ├── main.py               # tareas asyncio: adquisición, procesamiento, publicación
│   ├── settings.example.py   # settings.py real en .gitignore (SSID, PASS, BROKER, NODE_ID, SENSOR...)
│   ├── pymakr.conf
│   └── lib/
│       ├── mqtt_as.py
│       ├── mqtt_local.py     # corregido para Pico 2W
│       ├── sensores/
│       │   ├── mpu6050.py    # driver en uso: init, FIFO, lectura
│       │   └── adxl345.py    # alternativa, con la misma interfaz
│       └── procesamiento.py  # ventanas, RMS, pico, golpes
├── servidor/                 # estructura como docker_iot
│   ├── compose.yaml          # perfil local (expo/taller)
│   ├── compose.vps.yaml      # agrega swag y TLS (opcional)
│   ├── .env.sample
│   ├── mosquitto/config/mosquitto.conf
│   ├── influxdb/             # volumen de datos (en .gitignore)
│   ├── clienteMqtt/          # Dockerfile, clienteMqtt.py (aiomqtt + influxdb-client), requirements.txt
│   ├── web/                  # Flask + gunicorn: croquis SVG, /api/ultimo, sesiones
│   ├── simulador/            # nodo simulado (aiomqtt)
│   └── telegrambot/          # opcional (E4)
├── analisis/                 # scripts/notebooks numpy (FFT, umbrales)
├── datos/                    # exportaciones de pruebas (CSV)
└── docs/
    ├── bitacora.md
    ├── decisiones.md
    ├── protocolo_pruebas.md
    └── informe/
```
Convención: un commit por avance concreto, con el hito en el mensaje (por ejemplo, `H3: conteo de golpes por factor de cresta`).

---

## 8. Hitos

Cada hito tiene un **objetivo**, **tareas**, la **evidencia** que tengo que aportar y un **criterio de aceptación**. No se pasa al siguiente hito hasta cumplir el criterio, salvo decisión explícita registrada en `decisiones.md`.

**Camino crítico para la expo:** H0 → H1 → H2 → H3 → H4 → H5 → H6 → H7 → H8 → H10 → H11. H9 y las extensiones dependen del tiempo. H5 y H6 se pueden avanzar en paralelo con H1–H4 usando el simulador.

### Fase A: Preparación
**H0. Puesta en marcha**
- Tareas:
  - Enviar el tema a la cátedra para su aprobación.
  - Crear el repositorio nuevo con la estructura de la sección 7 y este archivo como `CLAUDE.md`.
  - Verificar el entorno: VS Code con la extensión de Pico y Claude Code; Docker en la PC.
  - **Verificar la versión de MicroPython** (que sea firmware para RP2350 / Pico 2W).
  - **Verificar si la VPS de la materia sigue activa** y si es accesible desde fuera de la facultad (por el dominio de DuckDNS o por WireGuard).
  - Soldar los pines del sensor y de la Pico.
- Evidencia: mail enviado (y la respuesta), enlace al repositorio, salida de `import sys; sys.implementation`, estado de la VPS, foto del montaje.
- Criterio: tema enviado, repositorio con el primer commit, MicroPython correcto en la Pico y estado de la VPS conocido.

### Fase B: Nodo (firmware)
**H1. Sensor verificado**
- Tareas: cablear el MPU6050 por I2C; `i2c.scan()`; leer WHO_AM_I; configurar ±16 g, DLPF de 184 Hz y 1000 Hz; leer X/Y/Z.
- Evidencia: salida de la consola, incluido el valor de WHO_AM_I (anotar si es un clon).
- Criterio: aparece 0x68 en el scan, WHO_AM_I responde y, con el sensor quieto y plano, Z ≈ 1 g y X, Y ≈ 0 g. Golpecitos visibles como picos.

**H2. Adquisición continua sin pérdidas**
- Tareas: driver `lib/sensores/mpu6050.py` (interfaz común de la sección 4) con FIFO solo del acelerómetro; tarea asyncio que lee por bloques de 6 bytes; detección y recuperación de desbordes; medir la tasa efectiva.
- Evidencia: muestras por segundo medidas durante 60 s y cantidad de desbordes del FIFO.
- Criterio: tasa efectiva dentro de ±2 % de la configurada y cero desbordes en 60 s.

**H3. Procesamiento en el nodo**
- Tareas: ventanas de ~500 ms; quitar la gravedad; RMS por eje y total; pico; conteo de golpes.
- Evidencia: tabla o gráfico de tres casos: sensor quieto, vibración continua (ventilador, celular) y golpes con un destornillador.
- Criterio: los tres casos se distinguen claramente: RMS bajo; RMS alto con pocos golpes; golpes altos.

**H4. Conectividad**
- Tareas: integrar `mqtt_as` (como en clase) con `settings.py`; corregir `mqtt_local.py` para Pico 2W; NTP; Last Will en `estado`; publicar con el contrato de la sección 6, **sin romper la adquisición** (`ovf` = 0).
- Evidencia: `mosquitto_sub -v -t 'cirfren/#'` (o MQTTX) mostrando los mensajes; prueba de cortar y volver a dar WiFi.
- Criterio: mensajes válidos según el contrato, `seq` sin saltos y `ovf` = 0 en condiciones normales, reconexión sola en menos de 30 s y `estado` pasa a `offline` al desconectar.

### Fase C: Servidor
**H5. Backend en contenedores**
- Tareas: `compose.yaml` local con Mosquitto (usuario y contraseña), **InfluxDB 2.x** (organización `cirfren`, bucket `vibraciones` y token con permisos solo para ese bucket, guardado en `.env`), clienteMqtt adaptado (aiomqtt + influxdb-client, esquema de la sección 6) y simulador.
- Evidencia: `docker compose up -d` levanta todo; el Data Explorer de InfluxDB (:8086) muestra puntos del simulador y del nodo real, separados por el *tag* `node`.
- Criterio: con un solo comando se levanta el sistema y se guardan datos de dos zonas (una real y una simulada) asociados a una sesión.

### Fase D: Visualización
**H6. Frontend MVP**
- Tareas: app Flask + gunicorn (como el CRUD) con el croquis SVG del tren delantero (y el trasero preparado); endpoint `/api/ultimo`; JavaScript que actualiza los colores cada 0,5–1 s; panel de indicadores; inicio y fin de sesión con patente.
- Evidencia: captura o video desde el celular.
- Criterio: al golpear el sensor real, su zona cambia de color en el celular en menos de 1,5 s.

**H7. Integración extremo a extremo (banco de trabajo)**
- Tareas: nodo real + simulador + backend + web funcionando juntos; dashboard de Grafana (fuente de datos InfluxDB con Flux) con el historial por zona y una variable para elegir la sesión; exportación a CSV (desde Grafana o con una consulta Flux).
- Evidencia: video corto de todo el flujo; CSV de una sesión; captura de Grafana.
- Criterio: una sesión completa queda registrada y se puede revisar en Grafana o exportar para analizar.

### Fase E: Campo
**H8. Prueba en el taller**
- Tareas: escribir con Gustavo `protocolo_pruebas.md` (qué prueba y qué maniobra para cada síntoma); montaje provisorio (silicona sobre una plaquita, precintos o imán; Pico y powerbank en una bolsa hermética); calibración; **medición por turnos** izquierda/derecha en el banco de suspensión; si es posible, con un auto con falla conocida.
- Evidencia: CSV de cada pasada, fotos del montaje, comentarios de Gustavo.
- Criterio: al menos un auto medido de los dos lados con el mismo protocolo, y una conclusión registrada (coincide o no con el diagnóstico del mecánico).

**H9. Ajuste de umbrales y reglas** *(si hay tiempo)*
- Tareas: análisis con numpy de los datos de H8; definir los umbrales de color y las reglas de mensajes; opcionalmente, una FFT para ver en qué banda está la diferencia.
- Evidencia: script o notebook en `analisis/` con gráficos.
- Criterio: umbrales justificados con datos reales y documentados en `decisiones.md`.

### Fase F: Expo y entrega
**H10. Demo para la expo**
- Tareas: maqueta (tabla o tren dibujado con dos puntos de medición y un motorcito vibrador o golpes que el público pueda hacer); perfil local sin internet (notebook + hotspot); video de la prueba en el taller; ensayo cronometrado.
- Criterio: la demo funciona desde cero, sin internet, en menos de 5 minutos de preparación.

**H11. Presentación e informe**
- Tareas: póster o slides; guion oral (problema → solución → arquitectura → resultados → límites → trabajo futuro); README del repositorio; informe para la cátedra (problema, estrategia, resultados y código); mención obligatoria de que es un trabajo de la materia.
- Criterio: material listo y ensayado; informe y repositorio entregables.

### Extensiones (después del MVP, en orden sugerido)
- **E1.** Segundo nodo real (reemplaza al simulador) → medición simultánea.
- **E2.** Caja estanca, placa soldada y montaje con imán.
- **E3.** Despliegue en la VPS con TLS (swag, DuckDNS, Mosquitto en el 8883) y prueba en ruta.
- **E4.** Bot de Telegram (python-telegram-bot + influxdb-client + matplotlib) con el informe y el gráfico al cerrar la sesión.
- **E5.** Respaldo store-and-forward en la Pico.
- **E6.** Espectro (FFT) en la web o en Grafana.
- **E7.** Login en la web (como en el CRUD) y publicación detrás de nginx/swag.
- **E8.** Temperatura por rueda con MLX90614.
- **E9.** Historial por patente y comparación antes/después de la reparación.
- **Trabajo futuro (no se implementa):** clasificador de fallas con aprendizaje automático, entrenado con el dataset que genera el sistema.

---

## 9. Organización del trabajo

Hay dos herramientas, y cada una sirve para algo distinto:

### Claude Code en VS Code: todo el código
Se trabaja dentro del repositorio, y Claude lee `CLAUDE.md` automáticamente. Se usa para **firmware, servidor y web**: escribir y editar archivos, ejecutar comandos y hacer commits. Conviene abrir una sesión por área o por hito (por ejemplo, "H2 firmware" o "H5 servidor") en lugar de una sesión eterna. Al cerrar cada sesión, Claude actualiza la sección 11 y agrega la entrada en `docs/bitacora.md`.

### Proyecto en Claude.ai: lo que no es código
| Chat | Para qué | Hitos |
|---|---|---|
| **00 · Coordinación** | Planificar, revisar el estado, decidir cambios de alcance o del contrato. Empezar acá cada vez que no esté claro qué sigue. | Todos (seguimiento) |
| **01 · Pruebas y análisis** | Protocolo con Gustavo, calibración, interpretación de datos reales, umbrales y reglas. | H8, H9, E3 |
| **02 · Informe y presentación** | Mail a la cátedra, póster o slides, guion, informe, README, ensayo. | H0 (mail), H10, H11 |

**Reglas para que todo quede sincronizado:**
- La interfaz entre módulos es el **contrato de la sección 6**. Ningún chat ni sesión lo cambia por su cuenta.
- El `CLAUDE.md` del repositorio es la versión vigente. Cuando cambia, se vuelve a subir al proyecto de Claude.ai.
- Al cerrar una sesión de trabajo, Claude da un **resumen de cierre** (ver la sección 10).

---

## 10. Metodología de trabajo

### Ciclo por hito
1. **Arranque:** indico el hito en el que estoy (o Claude lo toma de la sección 11). Claude recuerda el objetivo y el criterio de aceptación.
2. **Plan corto:** Claude propone los pasos concretos del hito (sin adelantar hitos futuros salvo que se lo pida).
3. **Ejecución:** yo pruebo; Claude da código completo en archivos y explica lo necesario.
4. **Reporte:** publico los resultados con la plantilla de abajo.
5. **Verificación:** Claude contrasta el resultado con el criterio de aceptación. Si no se cumple, se diagnostica y se repite.
6. **Cierre:** resumen de cierre, commit, actualización de `bitacora.md` y de la sección 11.

### Plantilla para reportar avances o dudas
```markdown
### [AAAA-MM-DD] · H<n> · <título corto>
**Qué hice:** ...
**Resultado:** (salida de consola, datos, capturas, fotos)
**Funcionó / no funcionó:** ...
**Dudas o problemas:** ...
**Próximo paso que creo que sigue:** ...
```

### Resumen de cierre (lo genera Claude al terminar cada sesión)
```markdown
### [AAAA-MM-DD] · Cierre de sesión · <Claude Code | chat nn>
**Hito:** H<n>, cumplido / en progreso
**Logrado:** ...
**Decisiones tomadas:** ... (si cambian el contexto: texto propuesto para CLAUDE.md)
**Pendiente / próximo paso:** ...
```

### Registro de decisiones (`docs/decisiones.md`)
| Fecha | Decisión | Motivo | Alternativas descartadas |
|---|---|---|---|

### Instrucciones para Claude en este proyecto
- Responder en **español rioplatense**, con un tono técnico y directo. Soy estudiante avanzado de Computación: no hace falta explicar lo básico de programación, pero sí lo de electrónica, señales y mecánica cuando corresponda.
- Ser **procedimental**: trabajar sobre el hito actual, pedir evidencia y no dar un hito por cumplido sin que se cumpla su criterio de aceptación.
- **Reutilizar lo visto en clase** (sección 3) antes de proponer herramientas nuevas. Si algo nuevo hace falta, justificarlo.
- Respetar las restricciones: **un solo desarrollador, presupuesto de estudiante y hardware disponible** (sección 4).
- Ser honesto con los límites técnicos (ruido de fondo, localización, diagnóstico automático). No prometer lo que el hardware no puede hacer.
- Código como archivos completos, coherentes con la estructura del repositorio y el contrato de datos. Secretos solo en `settings.py` y `.env`, nunca en el repositorio.
- Señalar cualquier cambio que afecte este archivo y proponer o aplicar el texto actualizado.
- Recordar la seguridad: el conductor no mira el celular, los cables lejos de piezas móviles y del escape, y el auto bien calzado en el elevador o el banco.

---

## 11. Estado actual

> Se actualiza al cerrar cada sesión.

- **Fecha de actualización:** 2026-11-05
- **Hito actual:** H0 (puesta en marcha)
- **Hardware disponible:** 1 × Pico 2W (MicroPython ya instalado), 1 × MPU6050 (GY-521), fuente. Sin caja ni imán todavía. El ADXL345 no se consiguió.
- **Aprobación del tema por la cátedra:** pendiente de envío.
- **Repositorio del proyecto:** sin crear.
- **VPS de la materia:** estado desconocido (hay que verificarlo).

| Hito | Estado |
|---|---|
| H0 Puesta en marcha | ⏳ En curso |
| H1 Sensor verificado | ⬜ |
| H2 Adquisición sin pérdidas | ⬜ |
| H3 Procesamiento en el nodo | ⬜ |
| H4 Conectividad | ⬜ |
| H5 Backend en contenedores | ⬜ |
| H6 Frontend MVP | ⬜ |
| H7 Integración extremo a extremo | ⬜ |
| H8 Prueba en el taller | ⬜ |
| H9 Ajuste de umbrales | ⬜ |
| H10 Demo para la expo | ⬜ |
| H11 Presentación e informe | ⬜ |

**Dudas abiertas:**
- ¿La VPS sigue activa después del cursado? ¿Es accesible desde fuera de la red de la facultad?
- ¿La cátedra exige algún formato particular para el informe?

**Bitácora resumida:**
- 2026-11-05: definición del proyecto, del alcance, de la arquitectura, de los hitos y de la forma de trabajo (chat inicial). Se revisó el material de clase y los repositorios del profesor, y se alinearon las decisiones con lo dictado (v2 del contexto).
- 2026-11-05: **decisión:** InfluxDB 2.x en lugar de MariaDB (datos de series temporales, consultas por ventana, integración con Grafana, escalabilidad a datos crudos). Registrar en `docs/decisiones.md` al crear el repositorio.
- 2026-11-08: **decisión:** el MVP usa el **MPU6050** porque el ADXL345 no se consigue a tiempo. Límite asumido: ancho de banda de ~184–260 Hz (sin detección de rodamientos). Driver con interfaz común para poder migrar. El segundo nodo también será un MPU6050.
