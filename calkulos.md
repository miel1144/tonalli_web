# Tonalli — Memoria de cálculo

Documento de referencia de **todas** las fórmulas de la aplicación: qué calcula,
cómo lo calcula, de dónde sale cada constante y qué error tiene cada supuesto.

Estado: fases A, B y F aplicadas. Las secciones marcadas *(fase D)* y
*(fase C)* describen limitaciones vigentes que se atienden más adelante.

---

## 0. Notación y unidades

| Símbolo | Magnitud | Unidad | Nota |
|---|---|---|---|
| $E$ | Iluminancia | lx (lux) | Flujo luminoso por unidad de superficie |
| $P$ | Potencia | W (watt) | Ritmo de consumo, **instantáneo** |
| $W$ | Energía | Wh, kWh | Potencia × tiempo. Es lo que factura CFE |
| $t$ | Tiempo | s, h | |
| $f_u$ | Factor de uso | adimensional, 0–1 | Ciclo de trabajo |
| $p$ | Precio de la energía | \$/kWh | Varía por bloque |

**La distinción que más importa en esta app:** watt y watt-hora no son lo mismo.
El watt es un ritmo; el watt-hora es una cantidad acumulada. Un foco de 40 W
encendido dos horas no «gastó 40 W», gastó **80 Wh**. La versión anterior de
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
     nivel de luz (§2.2)                 consumo mensual (§3.2)
           │                                     │
     histéresis (§2.3)                  motor tarifario CFE (§4)
           │                                ├── recibo bimestral (§4.4)
      notificación                          ├── precio marginal (§4.5)
                                            └── riesgo DAC (§6)
                                                     │
     CRONÓMETRO ──── energía ahorrada (§5.2) ────────┤
                            │                        │
                            ├──── dinero (§5.3) ◄────┘
                            ├──── CO₂ (§7)
                            └──── solar (§8)
```

Hay un solo punto de acoplamiento entre las dos ramas: el **precio marginal**.
El catálogo de aparatos determina en qué bloque tarifario cae el hogar, y ese
bloque es el que valúa cada kWh ahorrado.

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

$$W_{\text{mes}} = \frac{250 \cdot 24 \cdot 0.35}{1000} \cdot 30 = 63\ \text{kWh/mes}$$

Sin factor de uso daban 180 kWh/mes, o **360 kWh bimestrales de un solo aparato**
— más que el recibo completo de una casa promedio en Tijuana.

Una lectura útil del factor: de las 24 h conectado, el refrigerador consume el
equivalente a $24 \times 0.35 = 8.4$ h a plena potencia.

### 3.2 Consumo total

$$W_{\text{hogar}} = \sum_{i \in \text{activos}} \frac{P_i \cdot h_i \cdot f_{u,i}}{1000} \cdot d$$

Solo entran los aparatos con el interruptor activado. Para el recibo:

$$W_{\text{bimestre}} = W_{\text{hogar}} \times 2$$

### 3.3 Error residual

El factor de uso baja el error de ~450 % a un orden de 30 %, pero sigue siendo un
promedio. El ciclo real de un refrigerador depende de la temperatura ambiente,
de qué tan lleno esté y de su antigüedad: en agosto en Tijuana trabaja bastante
más que en enero.

La corrección definitiva (opción B, pendiente) es capturar el **kWh/año de la
etiqueta de eficiencia energética** —FIDE/CONUEE en México, Energy Guide en
Estados Unidos—, que es una cifra medida bajo protocolo normado
(NOM-015-ENER para refrigeradores, NOM-023-ENER para aire acondicionado) y ya
incorpora el ciclo real:

$$W_{\text{mes}} = \frac{W_{\text{año, etiqueta}}}{12}$$

---

## 4. Motor tarifario CFE

### 4.1 Temporada

CFE define seis meses consecutivos de verano por localidad. La app usa
mayo–octubre por defecto (Baja California) y es configurable.

$$\text{verano} = (\text{mes} \ge m_{\text{inicio}}) \wedge (\text{mes} \le m_{\text{fin}})$$

Verano y fuera de verano tienen **topes de bloque y precios distintos**.

### 4.2 Estructura de bloques

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

### 4.3 Reparto del consumo

Los topes son mensuales, así que se escalan por la duración del periodo:

$$C_j = L_j \cdot m$$

donde $L_j$ es el tope mensual del bloque $j$ y $m$ el número de meses (1 para
una proyección mensual, 2 para el recibo bimestral). El consumo se reparte de
abajo hacia arriba:

$$
W_j = \min\left(C_j,\ W_{\text{total}} - \sum_{k<j} W_k\right)
$$

$$\text{Energía} = \sum_j W_j \cdot p_j$$

### 4.4 Del subtotal al total

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

### 4.5 Precio promedio contra precio marginal

$$p_{\text{promedio}} = \frac{\text{Total}}{W_{\text{total}}}
\qquad
p_{\text{marginal}} = p_{j^*} \cdot \left(1 + \tfrac{\text{IVA}\%}{100}\right)$$

donde $j^*$ es el último bloque alcanzado. Son dos números muy distintos y sirven
para cosas distintas: el promedio explica el recibo, el marginal valúa el ahorro.

### 4.6 Ejemplo completo — recibo real de Tijuana

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

Para comparar: la versión anterior de la app usaba \$0.98/kWh plano para todo.

### 4.7 Precios de referencia y su vigencia

Los precios de CFE cambian mes a mes por un factor de actualización. Como la app
es offline, la estrategia es: **estructura de bloques empotrada** (es estable,
viene del acuerdo tarifario) más **precios de referencia con fecha visible** que
el usuario puede sobrescribir copiándolos de su recibo, que los imprime
desglosados.

Solo la tarifa **1A en verano** está marcada como verificada, porque sus precios
salen de un recibo real. Las demás muestran una insignia naranja pidiendo que se
capturen. Referencia empotrada: agosto 2026.

### 4.8 Dos tarifas, a propósito

La app guarda por separado la **tarifa configurada** y la **tarifa con la que Mi
Hogar calculó su proyección**. Cambiar de tarifa no reescribe la proyección sin
confirmación del usuario; mientras difieran, Mi Hogar muestra con cuál está
calculando y ofrece recalcular. La valuación del ahorro sí usa la configurada de
inmediato, porque ahí no hay un cálculo previo que preservar.

---

## 5. Sesión de ahorro

### 5.1 Tiempo

$$t = \text{now} - t_{\text{inicio}}$$

Se persiste el **instante de inicio**, no un contador. Android suspende los
timers de Dart cuando la app pasa a segundo plano y puede matar el proceso: un
contador incrementado cada segundo perdería tiempo de forma no determinista y una
sesión larga se perdería completa. El `Timer` de la app solo refresca la pantalla.

### 5.2 Energía no consumida

$$W_{\text{ahorrado}} = P_{\text{alumbrado}} \cdot \frac{t_{\text{segundos}}}{3600}\quad [\text{Wh}]$$

$P_{\text{alumbrado}}$ es la suma de la potencia de los focos que el usuario
apaga, configurable (40 W por defecto). El divisor 3600 convierte segundos a
horas: el resultado es energía.

### 5.3 Dinero

$$\$_{\text{ahorrado}} = \frac{W_{\text{ahorrado}}}{1000} \cdot p_{\text{marginal}}$$

**Por qué el precio marginal y no el promedio.** El kWh que dejas de consumir es
siempre el **último** del periodo de facturación, es decir el del bloque más caro
que alcanzas. Si un hogar está en excedente, cada kWh que ahorra le sale del
excedente, no del básico. Valuarlo al precio del bloque básico subestima el
ahorro real por un factor que puede ser de 3 o 4 — y eso juega en contra del
propósito de la app, porque le dice al usuario que ahorró mucho menos de lo que
ahorró.

Si el usuario todavía no registra aparatos, no se sabe en qué bloque cae y se usa
el **intermedio** como supuesto, con un aviso en pantalla.

**Ejemplo:** foco de 40 W apagado 3 h, hogar del §4.6.

$$W = 40 \cdot 3 = 120\ \text{Wh} = 0.12\ \text{kWh}$$
$$\$ = 0.12 \times 1.265 = \$0.152$$

---

## 6. Riesgo de tarifa DAC

$$r = \frac{W_{\text{hogar}}}{L_{\text{DAC}}}$$

| $r$ | Estado |
|---|---|
| $r < 0.85$ | Seguro |
| $0.85 \le r < 1.0$ | Cerca del límite |
| $r \ge 1.0$ | Rebasado |

**Lo que DAC no es.** No depende del monto del recibo. La versión anterior de la
app comparaba el costo contra \$1,500 — unidad equivocada, además de ventana
equivocada. CFE reclasifica el servicio cuando el **promedio móvil de los últimos
12 meses** de consumo mensual rebasa el límite de alto consumo de la tarifa de la
localidad.

$$\bar{W}_{12} = \frac{1}{12}\sum_{i=1}^{12} W_i > L_{\text{DAC}} \Rightarrow \text{DAC}$$

*(fase C)* Hoy la app compara la **proyección** del mes contra el límite, y lo
dice explícitamente en el aviso. El promedio real de 12 meses requiere histórico
y llega con la base de datos.

---

## 7. Emisiones evitadas

$$m_{\text{CO}_2} = \frac{W_{\text{ahorrado}}}{1000} \cdot FE$$

$$FE = 0.444\ \text{kg CO}_2\text{e/kWh}$$

Es el **Factor de Emisión del Sistema Eléctrico Nacional 2024**, publicado por la
Comisión Reguladora de Energía y notificado por SEMARNAT en el aviso del 28 de
febrero de 2025, para reporte al Registro Nacional de Emisiones.

Equivale a 0.444 tCO₂e/MWh. La versión anterior usaba 0.527, que corresponde a
años previos: el factor baja conforme entra generación renovable a la red.

**Ejemplo:** 0.12 kWh ahorrados → $0.12 \times 0.444 = 0.053$ kg CO₂e.

---

## 8. Dimensionamiento solar

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
fórmula anterior de la app lo omitía.

**Ejemplo** con el hogar del §4.6 (129 kWh/mes):

$$W_{\text{día}} = \frac{129}{30} = 4.3\ \text{kWh/día}$$
$$\text{Producción por panel} = 5.6 \times 0.55 \times 0.78 = 2.40\ \text{kWh/día}$$
$$N = \left\lceil \frac{4.3}{2.40} \right\rceil = \lceil 1.79 \rceil = 2\ \text{paneles} = 1.10\ \text{kWp}$$

Es un **predimensionamiento**. La interconexión doméstica tiene un tope
regulatorio y el balance neto se liquida por periodo de facturación, no instante
a instante.

---

## 9. Tabla de constantes

| Constante | Valor | Dónde vive | Fuente |
|---|---|---|---|
| Muestras de la media móvil | 5 | `SensorService` | Criterio de diseño |
| Umbral de entrada a alerta | 300 lx | `SensorService` | Heurística |
| Umbral de salida de alerta | 250 lx | `SensorService` | Heurística (banda muerta) |
| Persistencia de la alerta | 30 s | `SensorService` | Criterio de diseño |
| Cooldown entre avisos | 30 min | `SensorService` | Criterio de diseño |
| Días por mes | 30 | `AppConstants` | Convención |
| Meses por recibo | 2 | `AppConstants` | CFE factura bimestral |
| IVA frontera | 8 % | `AppConstants` | Región fronteriza norte |
| IVA general | 16 % | `AppConstants` | Resto del país |
| Factor de emisión | 0.444 kg/kWh | `AppConstants` | FE-SEN 2024, CRE/SEMARNAT |
| HSP Tijuana | 5.6 | `AppConstants` | Recurso solar local |
| Performance Ratio | 0.78 | `AppConstants` | Práctica de ingeniería FV |
| Potencia del módulo | 0.55 kW | `AppConstants` | Módulo comercial |
| Potencia de alumbrado | 40 W | configurable | Valor inicial |
| Factores de uso | ver §3.1 | `TipoCarga` | Ciclos de trabajo típicos |
| Tarifas y límites DAC | ver §4.2 | `TarifasCfe` | Acuerdos tarifarios CFE |

---

## 10. Lo que la app **no** calcula

Para que quede explícito en la documentación del proyecto:

- No distingue luz natural de artificial *(fase D)*.
- No usa la hora del día ni el amanecer/atardecer *(fase D)*.
- No calibra el sensor contra un luxómetro patrón *(fase D)*.
- No conoce el consumo real facturado: todo parte del catálogo declarado por el
  usuario *(fase C: captura de recibos y calibración)*.
- No calcula el promedio móvil real de 12 meses para DAC *(fase C)*.
- No calcula el DSAP: lo copia el usuario, porque su fórmula es municipal.
- No modela variación estacional del ciclo de trabajo de los compresores.
- No actualiza precios por sí sola: es offline por diseño.

---

## 11. Referencias

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
