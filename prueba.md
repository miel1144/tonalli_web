# Fase D — pasos de instalación

Además de copiar `lib/` y el `pubspec.yaml`, esta fase toca configuración
nativa de Android. Sin estos pasos el modo vigilancia no arranca y los botones
de las notificaciones no responden con la app cerrada.

---

## 1. Dependencia

```
flutter pub get
```

Si falla por la versión de `flutter_foreground_task`, corre
`flutter pub add flutter_foreground_task` y deja la que resuelva. Es la única
dependencia cuya versión exacta no pude verificar desde aquí, y su API cambia
entre versiones mayores: si el analizador se queja dentro de
`lib/logic/vigilancia_service.dart`, ese es el archivo a ajustar. Está aislado
a propósito para que ningún otro dependa de esa API.

---

## 2. AndroidManifest.xml

En `android/app/src/main/AndroidManifest.xml`.

### Permisos, antes de `<application>`

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
```

`FOREGROUND_SERVICE_SPECIAL_USE` es obligatorio desde Android 14. Google pide
justificar ese tipo al publicar: la razón es que el sistema corta los sensores
continuos a las apps en segundo plano y no existe un tipo de servicio para
«lectura de sensor ambiental».

### Servicio, dentro de `<application>`

```xml
<service
    android:name="com.pravera.flutter_foreground_task.service.ForegroundService"
    android:foregroundServiceType="specialUse"
    android:exported="false">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="Medición continua del sensor de luz ambiental para
                       recomendar el apagado de alumbrado innecesario." />
</service>
```

### Receptor de las acciones de notificación

Permite que los botones «Sí», «No» e «Iniciar ahorro» funcionen sin abrir la
app. También dentro de `<application>`:

```xml
<receiver
    android:name="com.dexterous.flutterlocalnotifications.ActionBroadcastReceiver"
    android:exported="false" />
```

---

## 3. Verificación

1. **Día y noche.** La tarjeta de Inicio debe decir «Es de día» o «Es de noche»
   con la hora correcta del amanecer o el atardecer. Para Tijuana, el 1 de
   septiembre: amanecer 6:23, atardecer 19:13, luz útil hasta las 19:38.
2. **Modo vigilancia.** Actívalo, apaga la pantalla y deja el teléfono boca
   arriba bajo una lámpara. Debe aparecer el aviso permanente «Tonalli está
   midiendo la luz». A los 30 segundos sostenidos por encima del umbral debe
   llegar la notificación con sus botones.
3. **Botón con la app cerrada.** Cierra la app por completo, dispara un aviso y
   toca «Iniciar ahorro». Vuelve a abrir: el cronómetro debe estar corriendo
   desde el momento en que tocaste el botón, no desde que abriste.
4. **Detección de bolsillo.** Con el modo activo, guarda el teléfono en la
   bolsa tres minutos. Los avisos deben quedar en pausa y la tarjeta de Inicio
   debe decirlo.

---

## 4. Umbrales que quedaron

| Situación | Umbral | Nivel |
|---|---|---|
| Noche, ≥ 35 lx | entrada 35 / salida 28 | Sugerencia |
| Noche, ≥ 150 lx | entrada 150 / salida 120 | Urgente |
| Crepúsculo, ≥ 35 lx | entrada 35 / salida 28 | Sugerencia |
| Día, ≥ 300 lx | entrada 300 / salida 240 | Pregunta |
| Cualquiera, < 2 lx sostenido 3 min | — | Bolsillo: avisos en pausa |

Persistencia antes de avisar: 30 s. Cooldown por tipo de aviso: 30 min.
Silencio tras responder «No» a la pregunta diurna: 4 h.

Todos viven en `lib/core/recomendaciones.dart` como constantes con nombre.
