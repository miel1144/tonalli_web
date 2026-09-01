# Tonalli — Fases A y F aplicadas

Reemplaza tu carpeta `lib/`, tu `pubspec.yaml` y agrega `analysis_options.yaml`
en la raíz del proyecto.

---

## 1. Archivos que debes BORRAR

```
lib/screens/habitaciones_screen.dart      203 líneas, 100 % comentadas
lib/screens/editar_habitacion_screen.dart 208 líneas, 100 % comentadas
lib/widgets/custom_card.dart              1 línea de comentario
lib/core/constans.dart                    reemplazado por core/constants.dart
```

Borra también, si tu IDE los conserva, cualquier `.dart` que no aparezca en la
lista de la sección 2.

Además ya vienen eliminados dentro de los archivos nuevos:

- los bloques comentados al pie de `main.dart`, `timer_logic.dart` y `storage_logic.dart`
- el botón **«Probar Notificación Push (Modo Jurado)»** de `home_screen.dart`
- los comentarios de pitch (`// PITCH: cámbialo a 10 segundos para deslumbrar al jurado`)
- el catálogo precargado de refri/TV/microondas, que inflaba la estimación desde el primer arranque
- las **36 llamadas a `withOpacity`**, deprecadas, ahora `withValues(alpha:)`
- el `//import 'habitaciones_screen.dart';` huérfano de `main_screen.dart`

---

## 2. Archivos del entregable

| Archivo | Estado |
|---|---|
| `analysis_options.yaml` | **nuevo** |
| `pubspec.yaml` | reescrito |
| `lib/core/constants.dart` | **nuevo** (renombrado desde `constans.dart`) |
| `lib/core/tarifas_cfe.dart` | **nuevo** — motor tarifario |
| `lib/core/normativa.dart` | **nuevo** — referencias con fuente |
| `lib/core/theme.dart` | limpiado |
| `lib/logic/storage_logic.dart` | reescrito + migración |
| `lib/logic/timer_logic.dart` | reescrito |
| `lib/logic/sensor_logic.dart` | sin cambios de lógica (fase B) |
| `lib/logic/notification_service.dart` | sin cambios (fase B) |
| `lib/models/electrodomestico_model.dart` | limpiado |
| `lib/main.dart` | limpiado + `allowRuntimeFetching = false` |
| `lib/screens/configuracion_screen.dart` | **nuevo** |
| `lib/screens/electrodomesticos_screen.dart` | reescrito |
| `lib/screens/perfil_screen.dart` | reescrito |
| `lib/screens/onboarding_screen.dart` | reescrito |
| `lib/screens/ahorro_screen.dart` | reescrito |
| `lib/screens/home_screen.dart` | reescrito |
| `lib/screens/splash_screen.dart` | limpiado |
| `lib/screens/main_screen.dart` | limpiado |
| `lib/widgets/custom_button.dart` | limpiado |
| `lib/widgets/custom_nav_bar.dart` | limpiado |

---

## 3. Pasos para que compile

1. Borra los cuatro archivos de la sección 1.
2. Copia `lib/`, `pubspec.yaml` y `analysis_options.yaml`.
3. Crea la carpeta `assets/google_fonts/` y coloca ahí, con estos nombres exactos:
   `Poppins-Regular.ttf`, `Poppins-Medium.ttf`, `Poppins-SemiBold.ttf`,
   `Poppins-Bold.ttf` (los descargas de fonts.google.com).
   Sin ellos la app arranca, pero al no poder descargar caerá al tipo de letra
   del sistema — que es exactamente el bug que estamos cerrando.
4. `flutter pub get` y `flutter analyze`.

---

## 4. Qué cambió en la lógica

### Unidades

`potencia × tiempo` es energía. Todo lo que decía «W» y era Wh ahora dice Wh, y
se muestra en kWh en cuanto pasa de 1000. La llave `watts_total` migra sola a
`wh_total` **sin convertir el número**: el valor guardado siempre fue correcto,
lo que estaba mal era la etiqueta.

### Motor tarifario (`core/tarifas_cfe.dart`)

- Las 8 tarifas domésticas (1, 1A–1F, DAC) con sus bloques y sus límites.
- Temporada verano / fuera de verano, configurable (mayo–octubre por defecto).
- Reparto del consumo entre bloques escalando los topes por los meses del
  periodo, que es como CFE los aplica.
- IVA (8 % frontera / 16 % resto) y alumbrado público como conceptos aparte.
- **Precio marginal**: el kWh que dejas de consumir es el último del periodo, o
  sea el del bloque más caro que alcanzas. El ahorro ahora se valúa así.

Los bloques de la **1A en verano** vienen de tu recibo: 200 kWh básico a
$1.010 y 58 kWh intermedio a $1.171 en un bimestre confirman topes mensuales de
100 y 50 kWh. Ese esquema está marcado como **verificado**. El resto de las
tarifas trae precios de referencia y aparece con una insignia naranja pidiendo
que el usuario copie los suyos.

### DAC

Ya no se compara contra $1,500. Se compara el consumo en **kWh/mes** contra el
límite de alto consumo de la tarifa (300 kWh/mes para tu 1A), con tres estados:
seguro, cerca (≥85 %) y rebasado. El aviso dice explícitamente que CFE evalúa el
promedio móvil de 12 meses y que esto es una proyección hasta que exista el
histórico.

### Cronómetro

Se persiste el instante de inicio y el tiempo se calcula contra
`DateTime.now()`. El `Timer` solo refresca la pantalla. Con eso el tiempo medido
es real aunque Android suspenda la app, y una sesión sobrevive a que el sistema
mate el proceso: al abrir de nuevo, sigue corriendo.

### CO₂ y solar

- Factor de emisión: **0.444 kg CO₂e/kWh** (FE-SEN 2024, CRE/SEMARNAT, aviso del
  28/02/2025). El 0.527 anterior es de años previos.
- Solar: se agrega **Performance Ratio 0.78** y se usan **5.6 HSP** de Tijuana en
  lugar del promedio nacional de 5.0. La fórmula anterior subdimensionaba ~25 %.

### Normativa

La tabla de iluminación se separó a `core/normativa.dart` y cada renglón lleva su
fuente. Se agregó la Tabla 1 de la NOM-025-STPS-2008 transcrita, y la pestaña de
vivienda muestra un aviso explicando que **no existe NOM mexicana de iluminación
para casa-habitación** y que esos valores son buena práctica, no obligación
legal. Se quitaron las referencias sueltas a UNE-EN sin contexto.

---

## 5. Bugs de fase B que cayeron de paso

Al reescribir los archivos se resolvieron: el botón «Saltar» que no guardaba la
bandera del onboarding, las sesiones de 0 segundos que inflaban el contador, el
borrado por índice del `Dismissible` (ahora por id), la falta de validación en
el alta de aparatos, el uso de `BuildContext` después de `await`, el mes de 30
días donde CFE factura bimestres, y el overflow del `Column` sin scroll en
Inicio.

**Siguen pendientes para la fase B:** histéresis del sensor, `detenerSensor()`
que nunca se llama, el id de notificación que colisiona, el permiso de
notificaciones pedido antes de la UI, el stream sin `onError`, y que «Mi Hogar»
no se refresque al cambiar la tarifa desde Perfil (el `IndexedStack` la mantiene
viva y nadie le avisa).
