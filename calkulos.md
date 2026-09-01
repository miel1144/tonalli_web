# Tonalli — Memoria de cálculo

Documento de referencia de **todas** las fórmulas de la aplicación: qué calcula,
cómo lo calcula, de dónde sale cada constante y qué error tiene cada supuesto.

Estado: fases A, B, C y F aplicadas. Las secciones marcadas *(fase D)* describen
limitaciones vigentes que se atienden más adelante.

---

## 0. Notación y unidades

| Símbolo | Magnitud | Unidad | Nota |
|---|---|---|---|
| $E$ | Iluminancia | lx (lux) | Flujo luminoso por unidad de superficie |
| $P$ | Potencia | W (watt) | Ritmo de consumo, **instantáneo** |
| $W$ | Energía | Wh, kWh | Potencia × tiempo. Es lo que factura CFE |
| $t$ | Tiempo | s, h | |
| $f_u$ | Factor de uso | adimensional, 0–1 | Ciclo de trabajo |
| $r$ | Residuo de calibración | kWh/mes | Consumo real menos estimado |
| $\alpha$ | Corrección de ciclo | adimensional, ≤1 | Ajuste a cargas cicladas |
| $p$ | Precio de la energía | \$/kWh | Varía por bloque |

**La distinción que más importa en esta app:** watt y watt-hora no son lo mismo.
El watt es un ritmo; el watt-hora es una cantidad acumulada. Un foco de 40 W
encendido dos horas no «gastó 40 W», gastó **80 Wh**. La versión original de
Tonalli calculaba bien pero rotulaba mal: mostraba «W» donde tenía Wh. Toda la
interfaz actual dice Wh, y cambia a kWh en cuanto el número pasa de 1000.

$$1\ \text{kWh} = 1000\ \text{Wh}$$

---

## 1. Panorama: cómo se encadenan los cálculos

```
        SENSOR                        CATÁLOGO DE APARATOS
     lecturas crudas                    potencia + horas + tipo
           │                                     │
     media móvil (§2.1)                  energía por aparato (§3.1)
           │                                     │
     nivel de luz (§2.2)                 consumo bruto (§3.2)
           │                                     │
     histéresis (§2.3)                          │◄──── RECIBOS DE CFE
           │                                     │      (consumo real)
      notificación                        calibración (§4)
           │                                     │
     resumen diario ────┐               consumo calibrado
                        │                        │
                        │              motor tarifario CFE (§5)
                        │                 ├── recibo bimestral (§5.4)
                        │                 ├── precio marginal (§5.5)
                        │                 └── riesgo DAC (§7) ◄── recibos
                        │                          │
     CRONÓMETRO ── energía ahorrada (§6.2) ────────┤
                        │                          │
                        ├───── dinero (§6.3) ◄─────┘
                        ├───── CO₂ (§8)
                        ├───── solar (§9)
                        └───── histórico SQLite (§10)
```

Hay dos puntos de acoplamiento. El **precio marginal**: el catálogo determina en
qué bloque tarifario cae el hogar, y ese bloque valúa cada kWh ahorrado. Y los
**recibos**: son la única fuente de consumo real, y de ellos salen tanto la
calibración del catálogo como el criterio DAC.

---

## 2. Medición de luz

### 2.1 Media móvil

El sensor entrega lecturas crudas que saltan mucho: la sombra de una mano, el
reflejo de la pantalla o una nube cambian el valor de un instante a otro.
Además, muchos teléfonos no reportan de forma continua sino en escalones
(0, 10, 40, 100, 320…), así que el número parpadea aunque la luz sea estable.

Lo que se muestra y lo que alimenta la decisión es la media de las últimas $n$
muestras:

$$\bar{E} = \frac{1}{n}\sum_{i=1}^{n} E_i \qquad n = 5$$

Es una media móvil simple sobre una cola de longitud fija: entra una muestra
nueva, sale la más vieja. Por eso la pantalla dice «Promedio de 5 lecturas».

*Constante:* `SensorService.muestrasPromedio = 5`.

### 2.2 Clasificación del nivel

| Rango de $\bar{E}$ | Etiqueta |
|---|---|
| $< 20$ lx | Ambiente oscuro. Requiere iluminación. |
| $20 \le \bar{E} \le 100$ lx | Luz tenue. Suficiente para descansar o circular. |
| $100 < \bar{E} \le 300$ lx | Luz suficiente para tus actividades. |
| $> 300$ lx | Hay más luz de la que pide la tarea. |

**Origen de los umbrales.** El 100 y el 300 coinciden con renglones de la Tabla 1
de la NOM-025-STPS-2008 (100 lx para circulación, 300 lx para aulas y oficinas).
El 20 corresponde al renglón de exteriores. Pero la coincidencia no es
fundamento: la NOM-025 regula **centros de trabajo**, y aquí se aplican a una
vivienda sin declarar la tarea que la persona realiza. Son heurísticas
razonables, no un criterio normativo. La corrección de fondo se describe en §2.4.

### 2.3 Histéresis y disparo de la notificación

Una máquina de dos estados con **banda muerta**:

$$
\text{estado} =
\begin{cases}
\text{en alerta} & \text{si } \bar{E} > 300 \text{ lx} \\
\text{fuera de alerta} & \text{si } \bar{E} < 250 \text{ lx} \\
\text{sin cambio} & \text{si } 250 \le \bar{E} \le 300
\end{cases}
$$

Al **entrar** en alerta arranca un temporizador de 30 s. Si la condición sigue
vigente cuando vence, se notifica. Si $\bar{E}$ cae por debajo de 250 lx antes,
el temporizador se cancela.

Además hay un **cooldown**: no se notifica dos veces en menos de 30 minutos.

$$t_{\text{notificar}} \ge t_{\text{última}} + 30\ \text{min}$$

*Por qué la banda muerta.* Con un solo umbral en 300 lx, una iluminancia que
oscila alrededor de ese valor cancela y reinicia el temporizador continuamente y
la alerta **nunca** dispara. Separar el umbral de entrada del de salida rompe ese
ciclo: una vez dentro, hay que bajar 50 lx para salir.

*Constantes:* entrada 300 lx, salida 250 lx, persistencia 30 s, cooldown 30 min.

### 2.4 Limitaciones de la medición *(se atienden en la fase D)*

1. **Un foco encendido también produce 300 lx.** El luxómetro mide iluminancia
   total y no puede separar luz natural de artificial. La recomendación actual
   supone implícitamente que el exceso viene del sol, cosa que la app no sabe.
   La app tampoco usa la hora del día en ninguna parte.
2. **Sin calibración.** La NOM-025 exige luxómetro calibrado por laboratorio
   acreditado, medición en el plano de trabajo y una metodología de puntos. El
   sensor va en la cara frontal del teléfono, sin calibrar y con respuesta
   espectral distinta en cada modelo. La lectura es **orientativa** y no
   sustituye un estudio normativo.
3. **Falta el factor de calibración ajustable** y las instrucciones de colocación
   (pantalla hacia arriba, en el plano de trabajo, sin sombra de la mano).
4. **El criterio defendible** no es «hay mucha luz natural» sino «la iluminancia
   medida supera el nivel que pide la tarea declarada». Ese sí se ancla en la
   Tabla 1 de la NOM-025 y funciona de día y de noche.

---

## 3. Consumo del hogar

### 3.1 Energía de un aparato

$$W_{\text{mes}} = \frac{P \cdot h \cdot f_u}{1000} \cdot d$$

donde $P$ es la potencia nominal en W, $h$ las horas conectado al día, $f_u$ el
factor de uso y $d = 30$ días. El resultado está en kWh/mes.

**El factor de uso es lo que hace válida la fórmula.** `potencia × horas` supone
que el aparato consume su potencia de placa todo el tiempo que está encendido.
Eso es cierto para cargas de potencia constante y falso para todo lo que tiene
termostato y compresor: un refrigerador está enchufado 24 h, pero su compresor
solo trabaja una fracción de ese tiempo.

| Tipo de carga | $f_u$ | Aplica a |
|---|---|---|
| Constante | 1.00 | Focos, TV, computadora, lavadora, microondas |
| Refrigeración | 0.35 | Refrigerador |
| Congelación | 0.40 | Congelador |
| Climatización | 0.60 | Aire acondicionado, minisplit |
| Bombeo | 0.15 | Bomba de agua |
| Calentamiento de agua | 0.30 | Boiler eléctrico |

**Ejemplo — refrigerador de 250 W conectado 24 h:**

$$W_{\text{mes}} = \frac{250 \cdot 24 \cdot 0.35}{1000} \cdot 30 = 63.0\ \text{kWh/mes}$$

Sin factor de uso daban 180 kWh/mes, o **360 kWh bimestrales de un solo aparato**
— más que el recibo completo de una casa promedio en Tijuana.

Una lectura útil del factor: de las 24 h conectado, el refrigerador consume el
equivalente a $24 \times 0.35 = 8.4$ h a plena potencia.

### 3.2 Consumo bruto

$$W_{\text{bruto}} = \sum_{i \in \text{activos}} \frac{P_i \cdot h_i \cdot f_{u,i}}{1000} \cdot d$$

Solo entran los aparatos con el interruptor activado. «Bruto» significa: lo que
declara el catálogo, antes de contrastarlo con la realidad.

### 3.3 Error residual

El factor de uso baja el error de un orden de 450 % a un orden de 30 %, pero
sigue siendo un promedio. El ciclo real de un refrigerador depende de la
temperatura ambiente, de qué tan lleno esté y de su antigüedad: en agosto en
Tijuana trabaja bastante más que en enero.

Ese error residual es justo lo que ataca la calibración de la §4.

---

## 4. Calibración contra recibos reales

Los recibos de CFE que el usuario captura son la **única fuente de consumo real**
de toda la app. Todo lo demás es estimación.

### 4.1 El residuo

Cada recibo se normaliza a un mes de 30 días, porque los periodos facturados no
tienen la misma duración:

$$W_{\text{real}} = W_{\text{recibo}} \cdot \frac{30}{\text{días facturados}}$$

$$r = W_{\text{real}} - W_{\text{bruto}}$$

### 4.2 Por qué NO un factor único

Lo obvio sería calcular $k = W_{\text{real}} / W_{\text{bruto}}$ y multiplicar
todo por $k$. No se hace, y la razón importa.

Ese escalar tendría que absorber **cinco errores distintos a la vez**: ciclos de
trabajo mal supuestos, horas mal declaradas, aparatos no registrados, cargas
fantasma en espera, y variación estacional. Es una ecuación con cinco incógnitas.
Y hace algo peor que ser impreciso: si el usuario olvidó registrar el boiler,
$k$ le echa la culpa al refrigerador. Las cifras por aparato quedan infladas de
forma silenciosa — justo las que el usuario mira para decidir qué cambiar.

La corrección es **asimétrica**:

### 4.3 Subestimación ($r > 0$): carga no identificada

El faltante no se reparte. Se registra como un renglón propio del catálogo,
«Consumo no identificado»:

$$W_{\text{calibrado}} = W_{\text{bruto}} + r$$

Con eso pasan tres cosas a la vez. El total cuadra con el recibo, que es lo que
necesitan el aviso DAC y el precio marginal. Las cifras por aparato **no se
tocan**, así que siguen significando lo que dicen. Y el hueco se vuelve
información accionable: un residuo grande está diciendo «faltan aparatos en tu
catálogo», que es un diagnóstico, no un defecto.

### 4.4 Sobreestimación ($r < 0$): corrección del ciclo

Aquí sí se corrigen parámetros, pero solo el más incierto. Que un LED de 12 W
esté 5 h es un dato duro; que el compresor de un refrigerador trabaje el 35 % del
tiempo es una suposición. Así que el ajuste cae **solo sobre las cargas
cicladas**:

$$W_{\text{ciclado}} = \sum_{i\ \text{ciclados}} \frac{P_i h_i f_{u,i}}{1000} d$$

$$\alpha = \text{clamp}\left(1 + \frac{r}{W_{\text{ciclado}}},\ 0.4,\ 1.0\right)
\qquad f_u' = f_u \cdot \alpha$$

### 4.5 Los tres candados

Sin estas guardas la calibración haría más daño que bien.

**Candado 1 — rango de razón.** Si
$W_{\text{real}} / W_{\text{bruto}} \notin [0.4,\ 2.5]$, no se calibra y se
avisa. Una razón de 4 no significa que el refrigerador consuma el cuádruple;
significa que el catálogo está prácticamente vacío.

**Candado 2 — integridad del catálogo.** Un recibo se descarta si algún aparato
fue creado o modificado **dentro** de su periodo facturado. Comparar el consumo
de esos meses contra la estimación de hoy daría un residuo falso. Esto es lo que
justifica las columnas `creado_en` y `modificado_en` de la tabla `aparatos`.

**Candado 3 — media ponderada.** No se usa el último recibo, sino todos los
válidos de la temporada, con más peso a los recientes:

$$w_i = \frac{1}{1 + m_i} \qquad
\bar{r} = \frac{\sum_i r_i \cdot w_i}{\sum_i w_i}$$

donde $m_i$ son los meses transcurridos desde el fin del periodo $i$. Un bimestre
con visitas en casa o dos semanas de vacaciones no debe arrastrar la calibración
por sí solo.

### 4.6 Separación por temporada

Con un solo recibo no se puede distinguir «mi refrigerador trabaja más de lo
supuesto» de «en agosto todo trabaja más». Por eso la calibración se calcula
**por separado** para verano y fuera de verano, clasificando cada recibo por el
punto medio de su periodo:

$$t_{\text{medio}} = t_{\text{inicio}} + \frac{\text{días}}{2}$$

Esto cierra directamente la limitación estacional que quedaba abierta en §3.3.
Con tres recibos ya hay señal; con seis se cubre un año.

### 4.7 Ejemplo numérico

Catálogo declarado:

| Aparato | $P$ | $h$ | $f_u$ | kWh/mes |
|---|---|---|---|---|
| Refrigerador | 250 W | 24 | 0.35 | 63.0 |
| Televisión | 120 W | 4 | 1.00 | 14.4 |
| Focos LED (4×12 W) | 48 W | 5 | 1.00 | 7.2 |
| Laptop | 65 W | 6 | 1.00 | 11.7 |
| **Bruto** | | | | **96.3** |

Recibo real: 258 kWh en 62 días.

$$W_{\text{real}} = 258 \cdot \frac{30}{62} = 124.8\ \text{kWh/mes}$$
$$r = 124.8 - 96.3 = +28.5\ \text{kWh/mes}$$
$$\text{razón} = \frac{124.8}{96.3} = 1.30 \in [0.4,\ 2.5]\ \checkmark$$

Como $r > 0$, se registra como consumo no identificado:

$$W_{\text{calibrado}} = 96.3 + 28.5 = 124.8\ \text{kWh/mes}$$

**Caso inverso.** Si el recibo hubiera sido de 80 kWh/mes:

$$r = 80 - 96.3 = -16.3, \qquad W_{\text{ciclado}} = 63.0$$
$$\alpha = 1 - \frac{16.3}{63.0} = 0.741 \Rightarrow f_u' = 0.35 \cdot 0.741 = 0.259$$

Refrigerador corregido: $63.0 \times 0.741 = 46.7$ kWh/mes. Las cargas constantes
quedan intactas en 33.3, y el total da $46.7 + 33.3 = 80.0$ kWh/mes. Cierra.

### 4.8 Lo que la calibración **no** resuelve

Los recibos arreglan el **agregado**, no la **atribución**. Después de calibrar
se sabe que la casa consume 124.8 kWh/mes y la proyección lo refleja, pero sigue
sin saberse con certeza cuánto de eso es el refrigerador.

Para eso hacen falta las etiquetas de eficiencia energética (FIDE/CONUEE en
México, Energy Guide en Estados Unidos), que dan el consumo anual medido bajo
protocolo normado —NOM-015-ENER para refrigeradores, NOM-023-ENER para aire
acondicionado— y ya incorporan el ciclo real:

$$W_{\text{mes}} = \frac{W_{\text{año, etiqueta}}}{12}$$

Recibos y etiquetas no compiten, se complementan: capturando la etiqueta de los
dos aparatos grandes, el residuo se calcula sobre un remanente mucho más chico y
mejor portado.

---

## 5. Motor tarifario CFE

### 5.1 Temporada

CFE define seis meses consecutivos de verano por localidad. La app usa
mayo–octubre por defecto (Baja California) y es configurable.

$$\text{verano} = (\text{mes} \ge m_{\text{inicio}}) \wedge (\text{mes} \le m_{\text{fin}})$$

Verano y fuera de verano tienen **topes de bloque y precios distintos**.

### 5.2 Estructura de bloques

Fuera de verano, todas las tarifas 1A–1F (y la tarifa 1 todo el año) comparten la
misma estructura: básico 0–75 kWh/mes, intermedio 76–140, excedente 141+.

En verano cada tarifa abre sus bloques según el clima de la localidad:

| Tarifa | Básico | Intermedio bajo | Intermedio alto | Límite DAC |
|---|---|---|---|---|
| 1 | 75 | 65 | — | 250 kWh/mes |
| **1A** | **100** | **50** | — | **300 kWh/mes** |
| 1B | 125 | 100 | — | 400 kWh/mes |
| 1C | 150 | 150 | 150 | 850 kWh/mes |
| 1D | 175 | 225 | 200 | 1,000 kWh/mes |
| 1E | 300 | 450 | 150 | 2,000 kWh/mes |
| 1F | 300 | 900 | 1,300 | 2,500 kWh/mes |

Las cifras son kWh **mensuales** que cubre cada bloque; lo que sobra cae en
excedente, que no tiene tope.

### 5.3 Reparto del consumo

Los topes son mensuales, así que se escalan por la duración del periodo:

$$C_j = L_j \cdot m$$

donde $L_j$ es el tope mensual del bloque $j$ y $m$ el número de meses (1 para
una proyección mensual, 2 para el recibo bimestral). El consumo se reparte de
abajo hacia arriba:

$$W_j = \min\left(C_j,\ W_{\text{total}} - \sum_{k<j} W_k\right)$$

$$\text{Energía} = \sum_j W_j \cdot p_j$$

### 5.4 Del subtotal al total

$$
\begin{align}
\text{Base} &= \text{Energía} + \text{Cargo fijo} \\
\text{IVA} &= \text{Base} \times \tfrac{\text{IVA}\%}{100} \\
\text{Total} &= \text{Base} + \text{IVA} + \text{DSAP}
\end{align}
$$

- **Cargo fijo:** solo existe en DAC (~\$142.41/mes de referencia). En 1–1F es cero.
- **IVA:** 8 % en la región fronteriza norte, 16 % en el resto del país. Tijuana
  es frontera; el recibo de referencia lo confirma.
- **DSAP / DAP:** Derecho de Servicio de Alumbrado Público. Es un cobro municipal
  con fórmula propia, **no se puede calcular** desde el consumo. El usuario lo
  copia de su recibo. No es un detalle menor: en el recibo de referencia
  representa más de un cuarto del total.

### 5.5 Precio promedio contra precio marginal

$$p_{\text{promedio}} = \frac{\text{Total}}{W_{\text{total}}}
\qquad
p_{\text{marginal}} = p_{j^*} \cdot \left(1 + \tfrac{\text{IVA}\%}{100}\right)$$

donde $j^*$ es el último bloque alcanzado. Son dos números muy distintos y sirven
para cosas distintas: el promedio explica el recibo, el marginal valúa el ahorro.

### 5.6 Ejemplo completo — recibo real de Tijuana

Datos: tarifa 1A, verano, 258 kWh en un bimestre (18 jun – 19 ago 2026).

| Bloque | Tope | kWh asignados | Precio | Importe |
|---|---|---|---|---|
| Básico | $100 \times 2 = 200$ | 200 | \$1.010 | \$202.00 |
| Intermedio | $50 \times 2 = 100$ | 58 | \$1.171 | \$67.92 |
| Excedente | ∞ | 0 | — | \$0.00 |

$$
\begin{align}
\text{Energía} &= 269.92 \\
\text{IVA }(8\%) &= 269.92 \times 0.08 = 21.59 \\
\text{DSAP} &= 105.58 \\
\text{Total} &= \mathbf{397.09}
\end{align}
$$

El recibo real dice \$397.77. La diferencia de \$0.68 es redondeo y ajustes
menores de CFE. **El modelo reproduce el recibo con 0.2 % de error.**

Los dos precios:

$$p_{\text{promedio}} = \frac{397.09}{258} = \$1.539/\text{kWh}$$
$$p_{\text{marginal}} = 1.171 \times 1.08 = \$1.265/\text{kWh}$$

Para comparar: la versión original de la app usaba \$0.98/kWh plano para todo.

### 5.7 Precios de referencia y su vigencia

Los precios de CFE cambian mes a mes por un factor de actualización. Como la app
es offline, la estrategia es: **estructura de bloques empotrada** (es estable,
viene del acuerdo tarifario) más **precios de referencia con fecha visible** que
el usuario puede sobrescribir copiándolos de su recibo, que los imprime
desglosados.

Solo la tarifa **1A en verano** está marcada como verificada, porque sus precios
salen de un recibo real. Las demás muestran una insignia naranja pidiendo que se
capturen. Referencia empotrada: agosto 2026.

### 5.8 Dos tarifas, a propósito

La app guarda por separado la **tarifa configurada** y la **tarifa con la que Mi
Hogar calculó su proyección**. Cambiar de tarifa no reescribe la proyección sin
confirmación del usuario; mientras difieran, Mi Hogar muestra con cuál está
calculando y ofrece recalcular. La valuación del ahorro sí usa la configurada de
inmediato, porque ahí no hay un cálculo previo que preservar.

---

## 6. Sesión de ahorro

### 6.1 Tiempo

$$t = \text{now} - t_{\text{inicio}}$$

Se persiste el **instante de inicio**, no un contador. Android suspende los
timers de Dart cuando la app pasa a segundo plano y puede matar el proceso: un
contador incrementado cada segundo perdería tiempo de forma no determinista y una
sesión larga se perdería completa. El `Timer` de la app solo refresca la pantalla.

### 6.2 Energía no consumida

$$W_{\text{ahorrado}} = P_{\text{alumbrado}} \cdot \frac{t_{\text{segundos}}}{3600}\quad [\text{Wh}]$$

$P_{\text{alumbrado}}$ es la suma de la potencia de los focos que el usuario
apaga, configurable (40 W por defecto). El divisor 3600 convierte segundos a
horas: el resultado es energía.

### 6.3 Dinero

$$\$_{\text{ahorrado}} = \frac{W_{\text{ahorrado}}}{1000} \cdot p_{\text{marginal}}$$

**Por qué el precio marginal y no el promedio.** El kWh que dejas de consumir es
siempre el **último** del periodo de facturación, es decir el del bloque más caro
que alcanzas. Si un hogar está en excedente, cada kWh que ahorra le sale del
excedente, no del básico. Valuarlo al precio del bloque básico subestima el
ahorro real por un factor que puede ser de 3 o 4 — y eso juega en contra del
propósito de la app, porque le dice al usuario que ahorró mucho menos de lo que
ahorró.

El bloque se determina con el consumo **calibrado** (§4), no con el bruto. Si el
usuario todavía no registra aparatos, se usa el bloque intermedio como supuesto,
con un aviso en pantalla.

**Ejemplo:** foco de 40 W apagado 3 h, hogar del §5.6.

$$W = 40 \cdot 3 = 120\ \text{Wh} = 0.12\ \text{kWh}$$
$$\$ = 0.12 \times 1.265 = \$0.152$$

### 6.4 Congelado del precio

Cada sesión guarda el $p_{\text{marginal}}$ vigente en el momento en que
terminó. Cambiar de tarifa después **no reescribe** lo ya ahorrado: el histórico
queda auditable y las gráficas no se deforman retroactivamente.

---

## 7. Riesgo de tarifa DAC

**Lo que DAC no es.** No depende del monto del recibo. La versión original de la
app comparaba el costo contra \$1,500 — unidad equivocada además de ventana
equivocada. CFE reclasifica el servicio cuando el **promedio móvil de los últimos
12 meses** de consumo mensual rebasa el límite de alto consumo de la tarifa.

### 7.1 Promedio móvil real

A partir de los recibos capturados, normalizando por días para que periodos de
distinta duración pesen lo que deben:

$$\bar{W}_{12} = \frac{\sum_i W_i}{\left(\sum_i d_i\right)/30}$$

donde $W_i$ son los kWh facturados y $d_i$ los días de cada recibo dentro de los
últimos 12 meses.

La app marca este número como **real** solo cuando
$\left(\sum d_i\right)/30 \ge 6$ meses. Por debajo de medio año la muestra no
representa lo que CFE evalúa sobre doce, así que cae a la proyección del catálogo
calibrado y lo declara en pantalla.

### 7.2 Evaluación

$$\rho = \frac{\bar{W}_{12}}{L_{\text{DAC}}}$$

| $\rho$ | Estado |
|---|---|
| $\rho < 0.85$ | Seguro |
| $0.85 \le \rho < 1.0$ | Cerca del límite |
| $\rho \ge 1.0$ | Rebasado |

**Ejemplo** con el recibo de referencia (258 kWh en 62 días, tarifa 1A):

$$\bar{W}_{12} = \frac{258}{62/30} = 124.8\ \text{kWh/mes}
\qquad \rho = \frac{124.8}{300} = 0.42 \Rightarrow \text{seguro}$$

Con un solo recibo la cobertura es de 2.1 meses, así que la app lo presenta como
proyección, no como promedio real.

---

## 8. Emisiones evitadas

$$m_{\text{CO}_2} = \frac{W_{\text{ahorrado}}}{1000} \cdot FE$$

$$FE = 0.444\ \text{kg CO}_2\text{e/kWh}$$

Es el **Factor de Emisión del Sistema Eléctrico Nacional 2024**, publicado por la
Comisión Reguladora de Energía y notificado por SEMARNAT en el aviso del 28 de
febrero de 2025, para reporte al Registro Nacional de Emisiones.

Equivale a 0.444 tCO₂e/MWh. La versión original usaba 0.527, que corresponde a
años previos: el factor baja conforme entra generación renovable a la red.

**Ejemplo:** 0.12 kWh ahorrados → $0.12 \times 0.444 = 0.053$ kg CO₂e.

---

## 9. Dimensionamiento solar

$$N_{\text{paneles}} = \left\lceil \frac{W_{\text{día}}}{\text{HSP} \cdot P_{\text{panel}} \cdot PR} \right\rceil$$

| Parámetro | Valor | Origen |
|---|---|---|
| HSP | 5.6 kWh/m²/día | Recurso solar de Tijuana, BC |
| $P_{\text{panel}}$ | 0.55 kW | Módulo comercial de referencia |
| $PR$ | 0.78 | Performance Ratio |

**Qué es el Performance Ratio.** Recoge las pérdidas reales del sistema:
conversión del inversor, caída de eficiencia del módulo por temperatura,
resistencia del cableado, suciedad sobre el vidrio y desviación de la orientación
óptima. Sin él, el arreglo queda alrededor de **25 % subdimensionado** — la
fórmula original de la app lo omitía.

**Ejemplo** con el hogar calibrado del §4.7 (124.8 kWh/mes):

$$W_{\text{día}} = \frac{124.8}{30} = 4.16\ \text{kWh/día}$$
$$\text{Producción por panel} = 5.6 \times 0.55 \times 0.78 = 2.40\ \text{kWh/día}$$
$$N = \left\lceil \frac{4.16}{2.40} \right\rceil = \lceil 1.73 \rceil = 2\ \text{paneles} = 1.10\ \text{kWp}$$

Es un **predimensionamiento**. La interconexión doméstica tiene un tope
regulatorio y el balance neto se liquida por periodo de facturación, no instante
a instante.

---

## 10. Histórico

Vive en SQLite empotrado: sin red, sin permisos, sin almacenamiento externo.

### 10.1 Racha de días

Días consecutivos con al menos una sesión de ahorro. Se cuenta hacia atrás desde
hoy, o desde ayer si hoy todavía no hay sesión — la racha no debe romperse antes
de que el día termine. Si no hay sesión ni hoy ni ayer, la racha es cero.

### 10.2 Resumen diario del sensor

Guardar cada lectura llenaría la base sin aportar nada, porque lo que interesa es
el perfil de luz del día. Se acumula una muestra por minuto y el promedio se
recalcula de forma incremental, sin conservar las muestras individuales:

$$\bar{E}_{n+1} = \frac{\bar{E}_n \cdot n + E_{\text{nueva}}}{n+1}$$

También se conserva el máximo del día.

### 10.3 Acumulados heredados

Los tres contadores de la versión con SharedPreferences (energía, dinero,
sesiones) no tienen fecha asociada, así que no pueden entrar a la tabla de
sesiones sin ensuciar rachas, gráficas y comparaciones por periodo. Viven en una
tabla aparte y se suman a los totales:

$$W_{\text{total}} = \sum_{\text{sesiones}} W_i + W_{\text{legado}}$$

### 10.4 Sellos de tiempo del catálogo

Cada aparato guarda `creado_en` y `modificado_en`. No son metadatos decorativos:
son lo que permite descartar un recibo cuyo periodo cruza un cambio de catálogo
(candado 2 de §4.5). Los aparatos migrados de la versión anterior reciben una
fecha centinela de 2000-01-01, porque usar la fecha de migración invalidaría
todos los recibos anteriores a ella — justo los que el usuario tiene a la mano.

---

## 11. Tabla de constantes

| Constante | Valor | Dónde vive | Fuente |
|---|---|---|---|
| Muestras de la media móvil | 5 | `SensorService` | Criterio de diseño |
| Umbral de entrada a alerta | 300 lx | `SensorService` | Heurística |
| Umbral de salida de alerta | 250 lx | `SensorService` | Heurística (banda muerta) |
| Persistencia de la alerta | 30 s | `SensorService` | Criterio de diseño |
| Cooldown entre avisos | 30 min | `SensorService` | Criterio de diseño |
| Intervalo de registro diario | 1 min | `SensorService` | Criterio de diseño |
| Días por mes | 30 | `AppConstants` | Convención |
| Meses por recibo | 2 | `AppConstants` | CFE factura bimestral |
| IVA frontera | 8 % | `AppConstants` | Región fronteriza norte |
| IVA general | 16 % | `AppConstants` | Resto del país |
| Factor de emisión | 0.444 kg/kWh | `AppConstants` | FE-SEN 2024, CRE/SEMARNAT |
| HSP Tijuana | 5.6 | `AppConstants` | Recurso solar local |
| Performance Ratio | 0.78 | `AppConstants` | Práctica de ingeniería FV |
| Potencia del módulo | 0.55 kW | `AppConstants` | Módulo comercial |
| Razón mínima de calibración | 0.4 | `CalibracionService` | Candado 1 |
| Razón máxima de calibración | 2.5 | `CalibracionService` | Candado 1 |
| $\alpha$ mínimo | 0.4 | `CalibracionService` | Cota inferior del ciclo |
| Cobertura mínima para DAC real | 6 meses | `HistorialService` | Criterio de diseño |
| Potencia de alumbrado | 40 W | configurable | Valor inicial |
| Factores de uso | ver §3.1 | `TipoCarga` | Ciclos de trabajo típicos |
| Tarifas y límites DAC | ver §5.2 | `TarifasCfe` | Acuerdos tarifarios CFE |

---

## 12. Lo que la app **no** calcula

Para que quede explícito en la documentación del proyecto:

- No distingue luz natural de artificial *(fase D)*.
- No usa la hora del día ni el amanecer/atardecer *(fase D)*.
- No calibra el sensor contra un luxómetro patrón *(fase D)*.
- No atribuye el consumo no identificado a aparatos concretos: sabe **cuánto**
  falta, no **de qué** (§4.8).
- No calcula el DSAP: lo copia el usuario, porque su fórmula es municipal.
- No modela variación dentro de una misma temporada, solo entre verano y fuera
  de verano.
- No actualiza precios por sí sola: es offline por diseño.

---

## 13. Referencias

1. **NOM-025-STPS-2008**, Condiciones de iluminación en los centros de trabajo.
   DOF 30/12/2008. Tabla 1, niveles mínimos de iluminación.
2. **CFE**, Tarifas de casa habitación (1, 1A–1F, DAC): estructura de bloques,
   temporada y límites de alto consumo.
3. **CRE / SEMARNAT**, Aviso del Factor de Emisión del Sistema Eléctrico
   Nacional 2024, publicado el 28 de febrero de 2025.
4. **Recibo CFE Tijuana**, periodo facturado 18 jun – 19 ago 2026, tarifa 1A.
   Fuente de los precios verificados y de la estructura de bloques de verano.
5. **NOM-007-ENER-2014**, eficiencia energética en alumbrado de edificios **no**
   residenciales. Se cita para dejar claro que **no aplica a vivienda**.
6. **NOM-015-ENER** y **NOM-023-ENER**, consumo energético de refrigeradores y de
   equipos de aire acondicionado. Base de la etiqueta de eficiencia.
