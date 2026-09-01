# Análisis técnico — Tonalli (Flutter / Android)

Auditoría de `lib/` (20 archivos, ~3,339 líneas), `pubspec.yaml` y `pubspec.lock`.
Dart 3.10.7 · Flutter ≥ 3.38.4 · Sin cambios aplicados al código.

---

## 0. Veredicto en una línea

La app **funciona y la idea es sólida**, pero hoy mide bien y *calcula mal*: hay un error de unidades que atraviesa toda la interfaz, el modelo tarifario no corresponde a cómo cobra CFE, el criterio de DAC está en la unidad equivocada, y no existe la estructura de datos mínima para construir el histórico que quieres como siguiente paso. Nada de eso es difícil de arreglar; todo es difícil de arreglar *después* de haber construido el histórico encima.

---

## 1. Mapa real de la aplicación

```
main.dart ──> SplashScreen ──> (onboarding_visto?) ──> OnboardingScreen | MainScreen
                                                            │
                                                            └─ IndexedStack (4 pantallas vivas siempre)
                                                                 0 ElectrodomesticosScreen  "Mi hogar"
                                                                 1 HomeScreen               "Inicio"   (luxómetro + botón ahorro)
                                                                 2 AhorroScreen             "Ahorro"   (cronómetro + stats)
                                                                 3 PerfilScreen             "Perfil"   (logros, CO₂, config)

Servicios (3 singletons con ChangeNotifier / addListener manual):
  StorageService       SharedPreferences  (5 llaves escalares + 1 StringList)
  SensorService        stream de ambient_light
  TimerService         Timer.periodic en RAM
```

**Código muerto:** `habitaciones_screen.dart` (203 líneas) y `editar_habitacion_screen.dart` (208 líneas) están **100% comentadas**; `custom_card.dart` es un solo comentario; `AppConstants` en `constans.dart` está comentado; hay bloques grandes comentados al pie de `main.dart`, `timer_logic.dart` y `storage_logic.dart`. En total ~500 líneas muertas sobre 3,339 (15% del proyecto).

---

## 2. Hallazgos — Nivel 1 (comprometen la validez del producto)

### 2.1 Error de unidades: watts-hora presentados como watts

`timer_logic.dart:53`

```dart
double get wattsAhorrados => (consumoFocosWatts * (_segundos / 3600.0));
```

`W × h = Wh`. El resultado son **watts-hora (energía)**, no watts (potencia). Pero la UI dice:

- `ahorro_screen.dart:212` — `"Energía (Watts)"`, valor `"... W"`
- `perfil_screen.dart:246` — `"${watts.toStringAsFixed(1)} W"`
- `perfil_screen.dart:153` — `logroEcoHeroe = watts > 100.0` (en realidad 100 Wh)
- `storage_logic.dart:19` — llave `watts_total`

Irónicamente el cálculo de CO₂ (`perfil_screen.dart:156`) *sí* asume Wh (`watts / 1000` para obtener kWh) — o sea, el código está bien y la etiqueta está mal. En una app cuyo argumento central es la energía, cualquier profesor o jurado con formación eléctrica lo detecta en el primer minuto. **Debe decir Wh (o kWh cuando pase de 1000).**

### 2.2 La tarifa CFE está modelada como precio plano; CFE no cobra así

`storage_logic.dart:22` — `tarifaCfe ?? 0.98` $/kWh, un único número que se usa para todo:

- `timer_logic.dart:54` — valuación del ahorro
- `electrodomesticos_screen.dart:146` y `:769` — costo mensual proyectado y costo por aparato

CFE cobra por **bloques escalonados** (básico / intermedio / excedente) y con **temporada** (verano vs. fuera de verano). Consecuencias concretas:

1. La proyección mensual de "Mi Hogar" es sistemáticamente errónea: subestima a los hogares que se van al bloque excedente y sobreestima a los de consumo bajo.
2. **El ahorro está mal valuado por naturaleza.** El kWh que dejas de consumir es siempre el *último* del periodo, o sea el del bloque más caro (marginal). Valuarlo a $0.98 cuando el usuario está en excedente (referencia 2026: ~$3.2/kWh en tarifa 1) subestima el ahorro real en un factor de ~3. Esto juega *en contra* del propósito de la app: le estás diciendo al usuario que ahorró mucho menos de lo que ahorró.
3. El valor por defecto de 0.98 no corresponde a ninguna tarifa completa; es aproximadamente el cargo del bloque básico de tarifa 1.

### 2.3 El umbral DAC está en pesos; DAC se define en kWh

`electrodomesticos_screen.dart:18`

```dart
final double _limiteTarifaDAC = 1500.0;   // pesos
...
bool alertaDACActiva = costoMensual >= _limiteTarifaDAC;
```

Según CFE, la clasificación DAC **no depende del monto del recibo**. Depende de que el **promedio móvil de consumo mensual de los últimos 12 meses** rebase el límite de alto consumo de la tarifa de la localidad:

| Tarifa | Límite de alto consumo |
|---|---|
| 1  | 250 kWh/mes |
| 1A | 300 kWh/mes |
| 1B | 400 kWh/mes |
| 1C | 850 kWh/mes |
| 1D | 1,000 kWh/mes |
| 1E | 2,000 kWh/mes |
| 1F | 2,500 kWh/mes |

Tres errores encadenados: unidad equivocada (pesos vs. kWh), ventana equivocada (mes actual vs. promedio de 12 meses) y ausencia del concepto de "tarifa de la localidad".

**Nota sobre Tijuana:** la asignación 1A–1F depende de la temperatura media mensual de verano (1A requiere ≥25 °C, 1F ≥33 °C). Tijuana no alcanza esos umbrales climáticos, así que lo más probable es que caiga en **tarifa 1, con límite de 250 kWh/mes** — un umbral mucho más estricto de lo que la app supone hoy. Hay sitios no oficiales que afirman que Tijuana es 1F; eso corresponde a Mexicali, no a Tijuana. **Esto hay que confirmarlo con un recibo real antes de codificarlo**, y de todos modos la tarifa debe ser *seleccionable por el usuario*, no fija.

### 2.4 `google_fonts` rompe el requisito de "sin internet"

`pubspec.yaml` declara `google_fonts: ^8.1.0` y `theme.dart` usa `GoogleFonts.poppinsTextTheme()`. Por defecto ese paquete **descarga los .ttf desde fonts.google.com en tiempo de ejecución** y los cachea. En la primera ejecución sin red, la app cae al tipo de letra por defecto del sistema y tu identidad visual se pierde. Es la contradicción más directa con tu requisito declarado. Solución: empaquetar Poppins como asset local y declararlo en `pubspec.yaml` (`fonts:`), o quitar el paquete.

### 2.5 El cronómetro pierde tiempo (y sesiones)

`timer_logic.dart:37` usa `Timer.periodic` incrementando un contador en RAM. Dos fallas:

- **Doze / pantalla apagada.** Android suspende o retrasa los timers de Dart cuando la app pasa a segundo plano. Una sesión de ahorro de 3 horas con el teléfono en el bolsillo contará bastante menos de 3 horas. El ahorro reportado será *menor* al real, de forma no determinista.
- **Muerte del proceso.** Si el SO mata la app (muy probable en 3 horas), se pierde la sesión completa: no hay estado persistido de "sesión en curso".

La solución correcta no es un servicio en primer plano (complica la app); es **guardar el `DateTime` de inicio en SharedPreferences y calcular `DateTime.now().difference(inicio)`** al reanudar. El cronómetro visual se convierte en un simple render de esa diferencia. Esto además hace la sesión recuperable tras un cierre.

### 2.6 No existe la estructura de datos para el histórico

`storage_logic.dart` guarda **tres contadores acumulados** (`watts_total`, `dinero_total`, `sesiones_total`) sin ninguna marca de tiempo. Con eso es **imposible** construir el histórico que quieres: no se puede responder "¿cuánto ahorré la semana pasada?", "¿mi consumo bajó respecto al mes anterior?", ni graficar nada.

Este es el hallazgo que condiciona el orden del trabajo: **el modelo de datos tiene que rediseñarse antes de la UI del histórico**, no después.

### 2.7 El modelo `watts × horas` es inválido para las cargas que más pesan

`electrodomesticos_screen.dart:139`. El catálogo por defecto (`storage_logic.dart:66`) trae "Refrigerador, 250 W, 24 h" → **180 kWh/mes solo del refrigerador**. Un refrigerador doméstico real consume del orden de 30–60 kWh/mes, porque su compresor trabaja con un ciclo de servicio del 25–40%, no continuo. Lo mismo aplica al aire acondicionado (1500 W en el catálogo sugerido).

`potencia × horas` es válido para cargas resistivas o de potencia constante (focos, plancha, TV). Para equipos con compresor y termostato hay que introducir un **factor de uso**, o mejor: capturar el **consumo anual en kWh de la etiqueta de eficiencia energética (FIDE / Energy Guide)**, que es el dato que el fabricante está obligado a publicar y que ya incorpora el ciclo real.

Con el catálogo actual, un hogar de ejemplo aparece con ~200 kWh/mes solo por refri + TV + microondas, lo que ya lo pone al borde del límite DAC de tarifa 1 — es decir, **la app alarma con datos inflados.**

---

## 3. Hallazgos — Nivel 2 (bugs funcionales concretos)

| # | Archivo:línea | Problema |
|---|---|---|
| 1 | `onboarding_screen.dart:51-56` | El botón **"Saltar" no guarda `onboarding_visto`**. El tutorial reaparece en cada arranque para quien lo salta. |
| 2 | `sensor_logic.dart:76` | **`detenerSensor()` nunca se llama** en toda la app. El stream del sensor queda vivo indefinidamente (agravado por el `IndexedStack`, que nunca destruye `HomeScreen`). Drenaje de batería continuo. |
| 3 | `sensor_logic.dart:56-73` | **Sin histéresis.** Con lux oscilando alrededor de 300 (nubes, sombra, movimiento), el `Timer` de 30 s se cancela y reinicia sin parar y la notificación **nunca dispara**. Y al cruzar hacia abajo se resetea `_notificacionEnviada`, así que en el caso contrario puede notificar repetidamente. Falta banda muerta (entrar >300, salir <250) y *cooldown*. |
| 4 | `notification_service.dart:44` | `id: DateTime.now().millisecond` devuelve **0–999**. Colisiones frecuentes: las notificaciones se sobrescriben entre sí. Debe ser `millisecondsSinceEpoch.remainder(100000)`. |
| 5 | `timer_logic.dart:24-33` | Se **guarda sesión aunque `_segundos == 0`**. Dos toques seguidos inflan `sesiones_total` y desbloquean la insignia "Iniciador" sin ahorro real. |
| 6 | `electrodomesticos_screen.dart:790` | `_eliminarAparato(index)` usa el índice capturado en el `builder` del `Dismissible`. Con varios borrados rápidos puede eliminar el elemento equivocado. Debe borrarse **por `id`**. Tampoco hay confirmación ni "deshacer". |
| 7 | `electrodomesticos_screen.dart` (todo) | La pantalla lee la tarifa en `build` pero **solo carga datos en `initState`**. Como el `IndexedStack` la mantiene viva, si cambias la tarifa en Perfil, **"Mi Hogar" sigue mostrando los cálculos viejos** hasta reiniciar la app. Bug de sincronización real y visible. |
| 8 | `electrodomesticos_screen.dart:522-535` | El campo de watts usa `TextFormField` con `key: Key(wattsSeleccionados.toString())` como truco para forzar rebuild. Si el usuario escribe watts a mano y luego toca el dropdown, se pierde lo escrito. Y `double.tryParse(val) ?? 0` permite **guardar un aparato de 0 W**. Sin validación de rangos (watts > 0, horas 0–24). |
| 9 | `perfil_screen.dart:106-130`, `electrodomesticos_screen.dart:567-580` | Uso de `context` **después de `await` sin verificar `mounted`**. Lint `use_build_context_synchronously`; riesgo real de excepción si el widget se desmonta durante el guardado. |
| 10 | `electrodomesticos_screen.dart:139` | El mes se modela como **30 días fijos**, pero CFE factura a los hogares en **bimestres** (60–61 días). El número que muestra la app no es comparable contra el recibo que el usuario tiene en la mano. |
| 11 | `sensor_logic.dart:30` | El stream **no tiene `onError` ni manejo de "dispositivo sin sensor de luz"**. En un equipo sin sensor la app se queda en `0 lux` / `"Cargando..."` para siempre, sin explicación. |
| 12 | `home_screen.dart:193-311` | `Column` con dos `Spacer` y **sin scroll**. Con fuente del sistema grande o pantalla chica → *overflow*. |
| 13 | `home_screen.dart:151-178` | Botón **"Probar Notificación Push (Modo Jurado)"** visible en producción. Junto con los comentarios `// PITCH: cámbialo a 10 segundos para deslumbrar al jurado` (`timer_logic.dart:40`, `sensor_logic.dart:59`) y "para que el jurado vea datos de inmediato" (`storage_logic.dart:64`), delata que es un prototipo de hackathon. Hay que sacarlo o esconderlo tras un modo desarrollador. |
| 14 | Varios | `TextEditingController` creados dentro de métodos y **nunca liberados** (`dispose`). Fuga menor pero repetida. |
| 15 | `main.dart:14` | El **permiso de notificaciones se pide antes de mostrar cualquier UI**, sin contexto. Tasa de aceptación baja y mala práctica. Debe pedirse dentro del onboarding, explicando para qué. |

---

## 4. Hallazgos — Nivel 3 (calidad, mantenibilidad, accesibilidad)

- **36 usos de `withOpacity`**, deprecado desde Flutter 3.27. Con Flutter 3.38 el analyzer llenará la consola de warnings. Reemplazo: `.withValues(alpha: …)`.
- **Sin `analysis_options.yaml` ni tests.** `flutter_test` está declarado pero no hay carpeta `test/`.
- **Estado global por singletons + `addListener`/`setState` manual.** Funciona a esta escala, pero `notifyListeners()` cada segundo redibuja pantallas completas (`HomeScreen`, `AhorroScreen`, `PerfilScreen` escuchan al mismo `TimerService`). Un `ValueListenableBuilder` acotado al widget del cronómetro evita repintar todo cada segundo.
- **`Electrodomestico.categoria` es un campo muerto**: siempre se guarda `'General'` y nunca se lee. O se usa (agrupar/filtrar) o se elimina.
- **Contraste insuficiente**: `AppColors.gray` (#8C8C8C) sobre `bgColor` (#FEFAE0) da ~3.0:1, por debajo del mínimo AA (4.5:1) para texto pequeño — y es el color de casi todos los textos secundarios. Todos los `fontSize` están fijos, sin respetar el escalado de texto del sistema.
- **Sin `Semantics`, sin modo oscuro** (irónico en una app sobre luz), **sin i18n** (todo hardcodeado).
- **Inconsistencia de marca**: el paquete se llama `luz_proyecto`, la app se llama Tonalli.
- **`constans.dart`** — falta la 't' de `constants`.
- **Plataforma**: `ambient_light` solo tiene implementación Android; iOS no expone el sensor de luz ambiental como API pública. Aun así `flutter_launcher_icons` declara `ios: true`. Conviene documentar que es Android-only por limitación de plataforma, no por decisión de alcance.

---

## 5. Fundamentación normativa: qué se sostiene y qué no

### 5.1 NOM-025-STPS-2008 (tu tercera liga, DOF 30/12/2008)

Tabla 1 — niveles mínimos de iluminación:

| Área / tarea | Lux mínimos |
|---|---|
| Exteriores generales: patios y estacionamientos | 20 |
| Interiores generales: almacenes de poco movimiento, pasillos, escaleras, iluminación de emergencia | 50 |
| Áreas de circulación y pasillos; salas de espera; salas de descanso; almacén | 100 |
| Requerimiento visual simple: inspección visual, recuento de piezas, trabajo en banco | 200 |
| Distinción moderada: ensamble simple, empaque, **aulas y oficinas** | 300 |
| Distinción clara: **salas de cómputo, áreas de dibujo, laboratorios** | 500 |
| Distinción fina: pintura, acabado de superficies, control de calidad | 750 |
| Alta exactitud: ensamble e inspección de piezas complejas | 1,000 |
| Alto grado de especialización | 2,000 |

**El problema de fondo:** NOM-025 aplica a **centros de trabajo**, no a viviendas. La NOM-007-ENER-2014 (que la misma fuente de GBR describe) aplica explícitamente a **edificios *no* residenciales**. **No existe una NOM mexicana que fije niveles de iluminación en casa-habitación.**

Consecuencia directa: la pestaña **"Residencial"** de tu modal `_estandaresIluminacion` (`electrodomesticos_screen.dart:24-56`) **no tiene fundamento normativo mexicano**, y las otras dos pestañas citan **UNE-EN 12464-1 y UNE-EN 12193**, que son normas **europeas** — y que, además, también son para lugares de trabajo, no para vivienda.

Esto no invalida los valores (son razonables y coinciden con la práctica de la ingeniería de iluminación), pero **sí invalida presentarlos como "normativa"**. Tienes dos salidas honestas, y la segunda es mejor:

1. Etiquetar cada renglón con su fuente real (NOM-025 donde aplique, UNE/CIE/IES donde no) y aclarar que para vivienda son *recomendaciones de buena práctica*, no obligación legal.
2. **Reestructurar la tabla alrededor de la tarea visual, no del cuarto.** Es lo que hace la propia NOM-025 y es conceptualmente más correcto: "leer" pide ~300–500 lux esté donde esté la persona. Así puedes anclar la mayoría de los renglones a la Tabla 1 con cita literal, y solo los ambientales (sala, dormitorio) quedan como recomendación referenciada.

### 5.2 Los umbrales del sensor no salen de ninguna norma

`sensor_logic.dart:40-52` usa 20 / 100 / 300 lux. El 100 y el 300 coinciden por casualidad con renglones de NOM-025; el 20 y la lógica de "más de 300 = luz natural, apaga el foco" son arbitrarios.

**La falla conceptual central de la app está aquí:** el luxómetro mide iluminancia total. **Un foco encendido también produce 300 lux.** La app no puede distinguir luz natural de artificial con un solo sensor, y sin embargo su recomendación principal ("Nivel excelente de luz natural, ¡apaga la luz artificial!") depende de esa distinción. Tú mismo describes el comportamiento deseado como *"si es de noche o de día"* — **pero la hora del día no se usa en ninguna parte del código**. A las 11 de la noche, con una lámpara de 400 lux, la app dice "Aprovecha el sol".

Tres formas de arreglarlo, de menor a mayor esfuerzo:

- **Hora del día** (`DateTime.now().hour`, o mejor, cálculo de amanecer/atardecer a partir de la latitud de Tijuana — es aritmética local, no requiere internet). De noche, el mensaje cambia de "aprovecha el sol" a "estás sobreiluminado para la tarea que realizas".
- **Delta al apagar.** Cuando el usuario toca "¡He apagado la luz!", compara los lux justo antes y justo después. Esa diferencia *es* la contribución del alumbrado artificial. Sirve para tres cosas a la vez: verificar que sí apagó algo, estimar la potencia real del foco sin que el usuario la capture, y calibrar el sensor.
- **Reencuadre del criterio**: en vez de "hay mucha luz natural", el criterio defendible es **"la iluminancia medida supera el nivel requerido por la tarea que declaró el usuario"** → hay sobreiluminación → hay margen de ahorro. Esto **sí** se ancla en NOM-025 (Tabla 1) y en el espíritu de NOM-007-ENER (no gastar más potencia de la necesaria), y funciona de día y de noche.

### 5.3 Limitación honesta del luxómetro del teléfono

NOM-025 exige luxómetro **calibrado por laboratorio acreditado**, medición en el plano de trabajo y una metodología de puntos. El sensor del teléfono está en la cara frontal, sin calibrar, con respuesta espectral distinta por modelo, y muchos equipos reportan en escalones (0, 10, 40, 100, 320…). La app debe: (a) declarar que la medición es **orientativa y no sustituye un estudio normativo**, (b) instruir cómo colocar el teléfono (pantalla hacia arriba, en el plano de trabajo, sin sombra de la mano), y (c) ofrecer un **factor de calibración** ajustable. Además conviene aplicar una **media móvil** de 5–10 muestras: hoy el número parpadea porque se pinta el valor crudo.

### 5.4 CFE (tu segunda liga)

Lo que la app debe modelar para ser defendible:

- **Tarifa seleccionable** (1, 1A–1F, DAC) — no un número plano.
- **Bloques** básico / intermedio / excedente, con sus límites en kWh y su cargo en $/kWh.
- **Temporada** verano / fuera de verano, que cambia tanto los límites de bloque como los cargos.
- **Periodo bimestral**, que es como llega el recibo.
- **Cargo fijo, DAP e IVA**, que son parte del importe pero no del costo de energía. Si no los modelas, dilo en la UI ("estimación de energía, antes de IVA y DAP").
- **DAC por promedio móvil de 12 meses en kWh**, no por monto.
- **Ahorro valuado al bloque marginal**, no al promedio.

Los cargos cambian mes a mes (CFE aplica un factor de actualización mensual). Como la app es offline, la estrategia correcta es: **valores de referencia empaquetados + fecha de vigencia visible + edición manual por el usuario** con instrucciones de dónde leerlos en su recibo. Nunca presentar el número como oficial y vigente sin fecha.

### 5.5 Factor de emisión de CO₂ — desactualizado

`perfil_screen.dart:157` usa `0.527 kg/kWh` etiquetado como "Factor de emisión oficial". El factor de emisión del Sistema Eléctrico Nacional que publica la CRE/SEMARNAT para reporte al RENE es **0.444 tCO₂e/MWh para 2024** (aviso del 28 de febrero de 2025). El 0.527 corresponde a años anteriores. Debe actualizarse, llevar año y fuente visible, y ser configurable — igual que la tarifa.

### 5.6 Calculadora solar — subdimensiona

`electrodomesticos_screen.dart:164-172`:

```dart
paneles = ceil( (kWh_mes / 30) / (HSP × 0.5 kW) )
```

Le falta el **factor de rendimiento del sistema (PR)**: pérdidas por inversor, temperatura, cableado, suciedad y orientación, típicamente 0.75–0.80. Sin él, el sistema queda **~25% subdimensionado**. La fórmula correcta:

```
paneles = ceil( (kWh_mes / 30) / (HSP_local × kW_panel × PR) )
```

Además: 5.0 HSP es un promedio nacional; **Tijuana ronda 5.5–5.8 kWh/m²/día**, y usar el valor local es justamente lo que hace defendible el cálculo para "el ciudadano tijuanense". Vale la pena mencionar en el modal que la interconexión doméstica tiene un tope regulatorio y que el balance neto se liquida por periodo de facturación, no instantáneamente.

---

## 6. Lo que hace falta para el histórico (el siguiente paso que quieres)

Hoy: 3 contadores escalares sin fecha. Lo mínimo para que el histórico sea posible:

```
Sesion { id, inicioISO, finISO, segundos, wh, pesos, luxPromedio?, luxAntes?, luxDespues? }
LecturaDiaria { fechaISO, luxPromedio, luxMax, minutosMedidos }
SnapshotMensual { anioMes, kwhEstimado, costoEstimado, tarifaUsada }
```

Con eso salen: racha de días, ahorro por semana/mes, gráfica de tendencia, comparación mes contra mes, promedio móvil de 12 meses para el aviso DAC honesto, y logros basados en constancia (que motivan mucho más que un umbral único de $2.00).

**Sobre el almacenamiento:** SharedPreferences con `setStringList` de JSON aguanta cientos de sesiones, pero es lectura/escritura del archivo completo cada vez y no admite consultas. Para un histórico con gráficas, la opción natural y **100% offline** es **`sqflite`** (SQLite empotrado, sin red, sin permisos, sin almacenamiento externo — cumple tu requisito). Alternativa más ligera: **Isar** o **Hive**. Es una decisión que conviene tomar *antes* de escribir la capa de histórico.

---

## 7. Plan propuesto y decisiones que necesito de ti

### Orden sugerido

| Fase | Contenido | Por qué en ese orden |
|---|---|---|
| **A. Corrección de fundamento** | Unidades Wh/kWh · modelo tarifario por bloques · DAC en kWh · factor CO₂ · factor solar PR + HSP local · fuentes normativas etiquetadas | Todo lo demás se construye encima. Cambiar unidades *después* del histórico obliga a migrar datos. |
| **B. Corrección de bugs** | Los 15 del Nivel 2 (empezando por Saltar, sensor sin detener, histéresis, delete por id, sincronización de tarifa) | Baratos, independientes entre sí, y varios son visibles al usuario. |
| **C. Rediseño del modelo de datos** | Sesión / lectura / snapshot + elección de motor (SQLite vs. Hive) + migración de los 3 contadores existentes | Habilita el histórico sin romper lo que ya guardó el usuario. |
| **D. Lógica del sensor v2** | Hora del día, media móvil, delta al apagar, calibración, criterio anclado a tarea | Es la mejora conceptual más valiosa; conviene hacerla con datos ya persistiéndose. |
| **E. Histórico + UX/UI** | Gráficas, tendencias, logros por constancia, rediseño visual, accesibilidad, modo oscuro | Tu paso final declarado. |
| **F. Limpieza** | Borrar 500 líneas muertas, `withOpacity`, `analysis_options.yaml`, quitar "Modo Jurado", tests | Puede ir en paralelo a cualquier fase. |

### Decisiones abiertas

1. **Tarifa de Tijuana.** ¿Tienes a la mano un recibo de CFE de Tijuana para confirmar si dice tarifa 1, 1A o 1C? Es el dato que ancla todo el módulo de costos, y prefiero verificarlo antes que codificar una suposición.
2. **Qué tan lejos llevar el modelo tarifario.** Tres niveles posibles: (a) bloques sin temporada, (b) bloques + verano/fuera de verano, (c) todo lo anterior + cargo fijo, DAP e IVA para que cuadre con el recibo. (b) es el mejor equilibrio esfuerzo/valor, pero es tu decisión.
3. **Motor de almacenamiento** para el histórico: SQLite (`sqflite`), Hive o seguir con SharedPreferences + JSON. Recomiendo SQLite.
4. **Modelo de electrodomésticos**: ¿introducimos factor de uso por categoría, o migramos a captura de kWh/año de la etiqueta de eficiencia? La segunda es más exacta pero le pide más al usuario.
5. **Alcance de la fase A**: ¿la ataco completa o prefieres empezar solo por las unidades y el DAC, que son los dos que más se notan en una revisión?
