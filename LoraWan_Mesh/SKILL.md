---
name: diseno-rf-lorawan-mesh
description: Diseño y verificación de radiofrecuencia para redes LoRaWAN con repetidoras mesh (Phantom Shield / RAK Gateway Mesh) en Colombia, para cualquier cliente o sitio. Úsala siempre que se hable de ubicar Border Gateway o repetidoras, altura de mástil o antena, cobertura, presupuesto de enlace, Fresnel, difracción, time-on-air, SF, dwell time, plan de frecuencias AU915, alcance entre nodos, pruebas de RF o PoC (RSSI, SNR, hop_count), o caídas de transmisión en medidores o sensores LoRa, aunque no se diga "diseño RF".
---

# Diseño RF LoRaWAN mesh

Sirve para dimensionar y revisar redes LoRaWAN de telemetría (medidores de energía, controladores de bomba contra incendio, sensores) en predios industriales, con un Border Gateway y repetidoras en malla SubGHz.

## Reglas de trabajo

- **Calcula con código, no estimes.** Todo número (ToA, FSPL, Fresnel, difracción, reservas) sale del módulo Python del final. Copia el módulo a un archivo y ejecútalo. No escribas cifras de memoria ni las reutilices de conversaciones anteriores: en un chat previo de este dominio una tabla de ToA omitió los 13 B de cabecera LoRaWAN y subestimó el airtime unos 90 ms.
- **No asumas datos de entrada.** Si falta una altura, una distancia, el payload o el SF, pídelo. Un supuesto solo es aceptable si lo marcas como tal y dices cuánto cambia el resultado si es otro.
- **Separa lo medido de lo supuesto.** Cada tabla o conclusión indica si viene de medición en campo, de una hoja de datos o de un supuesto de diseño.
- **Busca contradicciones en la oferta o el alcance** y señálalas aunque no te las pidan: tipo de backhaul (4G vs. LAN), número de repetidoras, tipo de antena en diagramas vs. texto, "no invasivo" vs. retiro de sellos, tasas de reporte distintas.
- **Responde solo lo que se pregunta**, en tablas cortas. Haz un informe o documento solo si te lo piden.

## Datos de entrada que necesitas

Pide los que falten antes de calcular algo que dependa de ellos:

1. Coordenadas o plano a escala con la ubicación del gateway, las repetidoras y cada nodo.
2. Alturas: antena del gateway, antena de cada repetidora (punto de fijación + mástil), nodo y cubierta de las naves cercanas.
3. Antena: ganancia y **apertura vertical (HPBW)** según el fabricante. Sin la hoja de datos, usa como supuesto 30° para 6 dBi y 18° para 8 dBi.
4. Radio: potencia TX, sensibilidad por SF, pérdidas de cable y conectores.
5. Payload útil en bytes por trama, periodo de reporte y número de nodos en el estado final, no solo en la fase actual.
6. Stack mesh: fabricante, overhead de cabecera, número máximo de saltos y si soporta más de un Border Gateway.
7. Encerramiento de cada nodo: tablero metálico, interior o exterior de la bodega, y fachada donde queda la antena.
8. Backhaul del gateway (LAN o 4G) y alimentación de cada repetidora.

Diseña sobre el **estado final** del proyecto aunque se instale por fases. Si se diseña solo la fase 1, lo normal es tener que reubicar después la infraestructura instalada.

## 1. Regulación (Colombia)

- Uso libre en **915–928 MHz** (ANE). Antes de citar la norma, búscala en la web para confirmar que sigue vigente y cuál es el límite de PIRE. En 2026 se conocían la Res. ANE 105 de 2020 y una reorganización del CNABF en 902–928 MHz que mantiene el uso libre en 915–928 MHz.
- Plan obligatorio: **AU915-928, sub-banda 2** (canales 8–15, uplink 916,8–918,2 MHz). Muchos gateways vienen de fábrica en US915 sub-banda 1 o 2 (902–905 MHz), que queda **fuera** de la banda de uso libre. Verifícalo en el equipo.
- **Dwell time de 0,4 s** por frecuencia. Por esto SF11 y SF12 no sirven para el diseño.

## 2. Airtime y techo de SF

Valores verificados con el módulo (BW 125 kHz, CR 4/5, preámbulo 8, CRC, 13 B de cabecera LoRaWAN y +14 B de cabecera mesh en uplink según RAK):

| SF | Payload útil máx. ≤400 ms, directo | Con mesh (+14 B) |
|---|---|---|
| SF8 | 125 B | 111 B |
| SF9 | 53 B | 39 B |
| SF10 | 11 B | **ninguno** |

Consecuencias:
- **Con mesh en SubGHz el techo operativo es SF9 y el payload útil máximo es 39 B.** SF10 no cumple el dwell time ni con payload vacío. El presupuesto de enlace se calcula con la sensibilidad de SF9.
- Si el stack no es RAK, pide su overhead real y recalcula.
- Cada salto retransmite el paquete: con N saltos, el tiempo en el aire es (N+1) veces el del enlace directo. Revisa la ocupación del canal con el número de nodos y el periodo de reporte. Con ALOHA puro, la probabilidad de colisión es P ≈ 1 − e^(−2G), donde G = nodos × ToA × (saltos+1) / (periodo × canales).
- Downlink Class A: RX1/RX2 a 1 s y 2 s. Pregunta cómo preserva el stack esas ventanas con varios saltos, porque de ellas dependen el join OTAA, el ADR y los comandos MAC.

## 3. Presupuesto de enlace

Reserva = pérdida admisible − pérdida de trayecto (dos rayos) − margen de desvanecimiento − obstrucciones.

- Pérdida admisible = PTX + Gtx + Grx − pérdidas − sensibilidad (SF9).
- Usa el modelo de dos rayos: espacio libre hasta el punto de ruptura (4·h1·h2/λ) y 40 dB/década desde ahí.
- Margen de desvanecimiento por defecto: 18 dB (supuesto de diseño; dilo).
- Criterio de reserva: **≥15 dB OK · 8–15 dB LÍMITE · <8 dB NO**.
- Despeje de Fresnel: 60 % de la primera zona libre de obstáculos en el trayecto.

El enlace crítico casi siempre es **nodo → repetidora** (nodo bajo, dentro de tablero, con naves de por medio). El backbone entre mástiles suele tener mucho margen. No gastes el análisis en el backbone.

## 4. Geometría: lo que cambia el diseño

**Cono de sombra de la antena omni.** Una omni gana ganancia estrechando el lóbulo vertical. Bajo un mástil alto, los nodos cercanos quedan fuera del lóbulo, en el nulo, con 15–25 dB de pérdida. La cobertura útil es una **corona**, no un círculo. Con 8 dBi (HPBW 18°) y el nodo a 1,5 m:

| Altura de antena | Borde −3 dB | Nulo por debajo de |
|---|---|---|
| 3 m | 9,5 m | 3,7 m |
| 10 m | 53,7 m | 21,0 m |
| 15 m | 85,2 m | 33,4 m |

Con mástiles altos, ninguna repetidora debe ir encima de las bodegas que atiende. Esa regla deja de aplicar en mástiles bajos (≤5 m).

**Altura vs. cubierta.** La antena debe quedar por encima de la línea de cubierta de las naves cercanas, o tener línea de vista por la vía. Si no, el punto no sirve, aunque la aritmética cierre.

**Bloqueo vehicular.** En mástiles bajos domina el bloqueo por tractomulas (~4,1 m) que cruzan la línea de vista cerca del nodo. Calcúlalo con filo de cuchillo y trátalo como intermitente, no como permanente.

**Difracción sobre naves.** Calcula con filo de cuchillo (ITU-R P.526) la nave más alta del trayecto. Revisa la fachada donde queda la antena del nodo: apuntar desde la fachada equivocada, a través de la propia bodega, puede costar más de 30 dB.

## 5. Reglas de arquitectura

Revisa cada una y señala las que el diseño o la oferta no cumplen:

1. **Energía en registro acumulado.** El payload debe llevar el registro acumulado de kWh, no el delta del periodo. Así, una trama perdida por bloqueo retrasa la facturación pero no pierde datos.
2. **Alarmas críticas sin salto mesh.** Sistemas cuyos eventos no se recuperan (RCI o bomba contra incendio, alarmas) van en enlace directo al Border Gateway, con uplink confirmado y heartbeat periódico.
3. **Alimentación de repetidoras:** circuito permanente con breaker propio y DPS. **Nunca del circuito de alumbrado**: la fotocelda o el horario lo apagan de día.
4. **Punto único de falla:** un solo Border Gateway lo es. Pregunta si el stack admite dos.
5. **Madurez del stack:** si el mesh está en beta (RAK Gateway Mesh lo estaba en 2026), es un riesgo para un sistema de facturación y debe informarse al cliente.
6. **Backhaul:** con LAN, el gateway queda amarrado al cuarto técnico. Con 4G puede ir al mejor punto de RF, pero hay que cotizar el plan de datos.
7. **Protección atmosférica** en mástiles y verificación de exposición RF: distancia de cumplimiento = √(PIRE / (4π·S)), con S = f(MHz)/200 W/m² (ICNIRP, público general).

## 6. Validación en campo (PoC)

- Pide por cada enlace: **RSSI, SNR, SF, frecuencia, hop_count, relay_id, fecha, hora y ubicación** de ambos extremos.
- **Verifica la procedencia del log**: sitio, fecha y distancia. Un log de otra ciudad o de otra fecha no valida este sitio.
- Si el nodo llega directo al gateway, la prueba **no demuestra el salto por la repetidora**. Para forzar el mesh, la distancia o la obstrucción deben impedir el enlace directo, o hay que bloquear el directo por configuración.
- Calibra el modelo con la diferencia entre la pérdida medida (PTX + ganancias − RSSI) y la calculada. Ajusta el margen o las obstrucciones, no la física.
- Barridos de espectro: haz un barrido de control con carga de 50 Ω para separar el piso de ruido del analizador del ruido real, y un max-hold de 24 h en 902–928 MHz.

## 7. Formato de respuesta

1. Conclusión directa (qué funciona, qué no y por qué) en 2–4 líneas.
2. Tablas con los resultados calculados, indicando el origen de cada dato.
3. Contradicciones o riesgos detectados.
4. Datos que faltan para cerrar el diseño, numerados.

## Módulo de cálculo

Copia este módulo a `rf.py` y úsalo para todos los números:

```python
import math
C = 299_792_458

def toa_ms(pl_app, sf, bw=125e3, lorawan_oh=13, mesh_oh=0, cr=1, pre=8, crc=1, ih=0):
    """Time-on-air (Semtech AN1200.13). pl_app = payload útil en bytes."""
    pl = pl_app + lorawan_oh + mesh_oh
    ts = (2 ** sf) / bw
    de = 1 if ts > 0.016 else 0
    n = 8 + max(math.ceil((8*pl - 4*sf + 28 + 16*crc - 20*ih) / (4*(sf - 2*de))) * (cr + 4), 0)
    return ((pre + 4.25) + n) * ts * 1000

def max_payload(sf, mesh_oh=0, limit_ms=400):
    ok = [p for p in range(256) if toa_ms(p, sf, mesh_oh=mesh_oh) <= limit_ms]
    return max(ok) if ok else None

def fspl(d, f=915e6):                    # d en m
    return 20*math.log10(d) + 20*math.log10(f) - 147.55

def breakpoint(h1, h2, f=915e6):
    return 4*h1*h2 / (C/f)

def pl_2ray(d, h1, h2, f=915e6):
    if d < breakpoint(h1, h2, f):
        return fspl(d, f)
    return max(fspl(d, f), 40*math.log10(d) - 20*math.log10(h1) - 20*math.log10(h2))

def fresnel_r(d1, d2, f=915e6, n=1):
    return math.sqrt(n * (C/f) * d1 * d2 / (d1 + d2))

def _J(v):                               # ITU-R P.526, filo de cuchillo
    return 0.0 if v <= -0.78 else 6.9 + 20*math.log10(math.sqrt((v-0.1)**2 + 1) + v - 0.1)

def knife_edge(d1, d2, h_obs, h_tx, h_rx, f=915e6):
    """Pérdida por un obstáculo de altura h_obs a d1 del TX y d2 del RX."""
    D = d1 + d2
    h = h_obs - (h_tx + (h_rx - h_tx) * d1 / D)
    return _J(h * math.sqrt(2*D / ((C/f) * d1 * d2)))

def shadow_cone(h_ant, h_node=1.5, hpbw_deg=18, null_deg=22):
    """Radio del borde -3 dB y radio bajo el cual el nodo cae en el nulo."""
    return ((h_ant - h_node) / math.tan(math.radians(hpbw_deg/2)),
            (h_ant - h_node) / math.tan(math.radians(null_deg)))

def reserve(budget_db, d, h_node, h_ant, obstruction_db=0, fade_db=18):
    r = budget_db - pl_2ray(d, h_node, h_ant) - fade_db - obstruction_db
    return r, ('OK' if r >= 15 else 'LIMITE' if r >= 8 else 'NO')

def collision_p(nodes, toa_s, period_s, channels=8, hops=0):
    G = nodes * toa_s * (hops + 1) / (period_s * channels)
    return 1 - math.exp(-2*G)

def rf_safety_distance(eirp_dbm, f_mhz=915):
    eirp_w = 10**(eirp_dbm/10) / 1000
    return math.sqrt(eirp_w / (4*math.pi*(f_mhz/200)))
```
