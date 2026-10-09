---
name: reveal-variables-electricas
description: Monitoreo e interpretación de variables eléctricas (tensión, corriente, potencia, energía, factor de potencia, frecuencia, THD, UPS, generador) en Reveal a través del MCP. Aplica a cualquier cliente. Úsalo cuando pregunten por medidores, consumo, calidad de energía, reglas o alertas eléctricas, datos que no llegan, o cuando aparezcan unit_id de magnitudes eléctricas en telemetría. Responde SOLO lo consultado, sin informes ni análisis generales no solicitados.
---

# Variables eléctricas en Reveal (genérico, independiente del cliente)

Evidencia levantada con el MCP de Reveal el 2026-10-09. Cada afirmación lleva un nivel de confianza:

- **[C] Confirmado**: devuelto por el MCP o documentado en los recursos `reveal://`.
- **[P] Probable**: deducido numéricamente de telemetría real; confirmar antes de reportar como hecho.
- **[R] Referencia de ingeniería**: conocimiento eléctrico general, no es configuración de Reveal. Ajustar a la norma y al sitio.
- **[?] Sin resolver**: no hay forma de confirmarlo con el MCP. Decirlo explícitamente.

---

## 0. Regla de alcance (prevalece sobre todo lo demás)

**Este documento es material de consulta, no un guion de informe.** Responder únicamente lo que el usuario preguntó, con la información mínima que lo contesta. Las secciones 3 a 9 se leen para resolver la pregunta; **no se vuelcan en la respuesta**.

Prohibido, salvo petición explícita del usuario:
- Entregar un estudio, auditoría o panorama general de electricidad del sitio o del medidor.
- Reportar variables que no se preguntaron (si preguntó tensión, no incluir corriente, potencia, FP, frecuencia ni energía).
- Revisar o listar reglas, overrides, alertas, relaciones o catálogo si la pregunta no los menciona.
- Explicar teoría, tolerancias, trampas del catálogo o limitaciones de herramientas que no cambian la respuesta.
- Añadir "hallazgos adicionales", recomendaciones o próximos pasos no solicitados.

### Qué llamar según la pregunta (solo eso)

| El usuario pregunta por… | Llamadas permitidas | Responder con |
|---|---|---|
| Un valor actual o reciente (tensión, corriente, kW, FP…) | `find_assets` si hay que resolver el nombre → `get_measurements` | Solo las variables nombradas: valor, unidad, `datetime_utc` |
| Un histórico, promedio, máximo o mínimo | `get_measurements` con la ventana pedida | Solo esa(s) variable(s) y esa ventana |
| Consumo de un periodo (kWh) | `get_measurements` del contador acumulado | `último − primero`, con unidad y ventana |
| Si un equipo reporta / no llegan datos | `get_current_state` | Estado de frescura y última lectura. Relaciones solo si el usuario pide la causa |
| Alertas o eventos | `get_alerts` / `get_events` | Solo las pedidas (sitio, equipo, periodo) |
| Una regla concreta o por qué no dispara | `get_component_rules`, y overrides solo si hace falta | Solo esa regla y la causa |
| Qué significa un unit_id | Catálogo (§4) | Una línea: nombre, unidad, nivel de confianza |

Si la pregunta no indica cliente, sitio, equipo o ventana y no se pueden deducir, **preguntar al usuario antes de consultar**. No recorrer todo el cliente para adivinar.

### Única excepción: advertencia puntual

Si al contestar se detecta algo que **invalida o distorsiona el dato que el usuario pidió** (por ejemplo, pidió potencia total y una fase tiene corriente con tensión ≈ 0), añadir **una sola línea** de advertencia pegada a ese dato y ofrecer ampliar. Cualquier otra observación se omite.

---

## 1. Principio rector

Una lectura eléctrica en Reveal **no trae nombre ni unidad**. Trae `site_component_id`, `unit_id`, `value`, `event_code` y `datetime_utc` [C]. Todo el significado depende de resolver `unit_id` contra el catálogo de Variables, y ese catálogo **no es universal ni completo vía MCP** (ver §4.4). Regla: **nunca afirmar qué mide un unit_id sin comprobarlo por catálogo y por magnitud física.**

## 2. Cadena de monitoreo

```
Medidor / UPS / generador (SiteComponent)
  → lecturas (datalog): N unit_id por componente, cada una con value y timestamp
  → motor de reglas evalúa cada lectura entrante contra reglas del tipo de componente
  → evento (pending → active → closed) ; si la regla está clasificada, es alerta con severidad real
```

- Un medidor publica muchas variables a la vez [C]. Observado: 6 unit_id (monofásico de bajo costo), 30 (trifásico tipo ME337) y 42 (trifásico con potencias por fase).
- La frecuencia de muestreo **varía por dispositivo** [C]: ≈1 lectura/min en dos medidores, ≈1 cada 25 min en otro (58 lecturas en 24 h). Calcularla antes de interpretar mínimos, máximos o ausencias.
- `datetime` (legado) coincide con UTC, no con hora local. Usar `datetime_utc`; `datetime_site` es hora local del sitio [C].

## 3. Componentes eléctricos del catálogo (component_id)

| Grupo | component_id (nombre) |
|---|---|
| Medición de energía | 88 Power Meter · 197 Power Meter Monofásico GSS · 122 Power Meter Congelados-Embutidos · 123 Power Meter PTAR · 163 Analizador de redes CVM-A1500 · 205 Power Monitoring Module · 208 Canal Power Meter (circuito) · 136 Medidor de corriente (Monnit) |
| Alimentación AC | 29 Alimentacion AC · 107 AC 208 · 108 AC 460 · 110 AC 460 Electrógeno |
| Respaldo | 87 UPS SNMP · 28 Batería (INC) · 150 Batería CSU · 203 Batería · 202 Rectificador |
| Generación | 109 Planta Electrógena · 204 y 201 Generador · 145 Power Station |
| Carga/actuadores | 121 Compressor · 46 AC (actuador de potencia) · 191 Actuador |
| Supervisión central | 148 Unidad de Supervisión Centralizada · 149 Sistema de Monitoreo Remoto CSU |
| PS (alimentación física) | 78 Power Source · 79 USB Claro |

Notas [C]: los tipos 201–206 están marcados `type="test"` en el catálogo aunque se usan en sitios reales. El inventario por sitio (`get_site_components`) puede venir truncado a 100 elementos (ver §9). Un componente eléctrico sin relaciones BRICS es invisible a `list_brick_*`.

Relaciones BRICS útiles para la cadena eléctrica [C]: `powers`, `suppliesFuelTo`, `sendsDataTo`, `controls`. Una arista `powers` desde un componente hacia otro indica dependencia de alimentación; úsala para explicar por qué varios equipos quedan sin datos a la vez.

## 4. Variables

### 4.1 Confirmadas en el catálogo [C] (`list_variables`)

| Magnitud | unit_id (nombre, unidad) |
|---|---|
| Tensión fase-neutro | 37 L1-N · 38 L2-N · 39 L3-N (V) · 74 Phase III Voltage (V) |
| Tensión fase-fase | 77 L1-L2 · 78 L2-L3 · 79 L1-L3 · 80 promedio L-L (V) |
| Corriente | 34 Fase 1 · 35 Fase 2 · 36 Fase 3 (A) · 59 Phase III Current (A) |
| Potencia activa | 40 · 41 · 42 por fase (kW) · 26 Input Total Active Power (kW) · 31 Output Active Power (W) · 67 Phase III Active Power (kW) |
| Potencia aparente | 4 Total Apparent Power (kVA) · 25 Input Total Apparent (kVA) · 32 Output Apparent (VA) · 69 Apparent Power (rotulada kVAh) |
| Potencia reactiva | 43 Total Reactive (kVAr) · 70 inductiva · 71 capacitiva (kVAr) · 91 (VAr) |
| Factor de potencia | 44 Total Power Factor · 72 Power Factor · 103 Total power factor · 66 Phase III inductivo |
| Frecuencia | 45 Frecuencia · 24 Input · 29 Output · 73 Phase 1 (Hz) |
| Energía | 5 kWh · 60 · 61 · 62 Imported Active Energy L1/L2/L3 (kWh) · 63 Phase III Active Energy (kWh) · 64 inductiva · 65 capacitiva (kWh) · 68 Apparent Energy (kVAh) · 3 (kVAh) · 90 Wh · 92 VArh · 104 Imported active energy Tariff 1 (Wh) |
| Distorsión | 81 · 82 · 83 THD corriente L1/L2/L3 · 84 THD corriente N (%) |
| UPS | 19 Capacidad batería (%) · 20 Temp. batería · 21 Autonomía restante (s) · 22 Voltaje batería (V) · 23 Input Line Voltage (VAC) · 28 Output Voltage (VAC) · 30 Output Current (A) · 33 Tiempo total en batería (s) · 27 Temp. interna |
| Generador/motor | 58 Generator Status · 85 y 95 Oil pressure · 86 Coolant temp · 98 Motor amperage · 99 FLA Motor · 100 Engine working hours · 101 kW est. motor |

### 4.2 Probables, observadas en telemetría real y **ausentes del catálogo devuelto** [P]

Deducidas porque `V·I·FP` reproduce el valor dentro de ≈1–2 % en un medidor trifásico (promedios 24 h parciales). **Confirmar contra el catálogo real de Reveal antes de usarlas en un informe.**

| unit_id | Interpretación probable | Respaldo |
|---|---|---|
| 109 · 110 · 111 | Tensión L1-L2 · L2-L3 · L1-L3 (V) | ≈211 V ≈ √3 × 121 V; las reglas 711–716 con esos nombres usan estos ids |
| 301 · 181 | Frecuencia (Hz) | ≈59,99 Hz; el unit_id 45 no apareció en las 3 muestras revisadas |
| 310 · 311 · 312 | Potencia activa por fase (W) | V·I·FP por fase coincide |
| 319 · 320 · 321 | Potencia aparente por fase (VA) | V·I por fase coincide |
| 316 · 317 · 318 | Potencia reactiva por fase (var, con signo) | √(S²−P²) del mismo orden |
| 313 · 314 · 315 | Potencia activa, aparente y reactiva totales | ≈ suma de las tres fases |
| 122 · 123 · 124 | Factor de potencia por fase | 122 reproduce P/S de la fase 1 |

### 4.3 Sin resolver [?]

Observados sin interpretación confirmable: 128–130 (≈2–2,5), 306–309, 117, 175, 361, 364, 182–189, 118. Reportarlos como "unit_id sin definición en el catálogo accesible", con rango observado. No inventar nombre.

### 4.4 Trampas del catálogo y de los unit_id [C]

- `list_variables` devolvió exactamente 100 filas (ids ≤104) mientras los medidores publican ids >300 y reglas usan 109–111, 379, 456, 459–461, 470–472. Es probable que la herramienta esté truncada a 100. Los ids >104 **no se pueden resolver desde el MCP**.
- **La misma magnitud tiene varios ids**: factor de potencia (44, 72, 103), frecuencia (24, 29, 45, 73, 181, 301), tensión L-L (77–79 y 109–111).
- **El mismo unit_id muestra magnitudes incompatibles entre medidores**: 117 ≈ 80 066 en uno y 0–1 en otro; 175 = 0 en uno y 112–168 en otro. No asumir semántica universal por unit_id; validar por tipo de medidor y magnitud.
- Rótulos inconsistentes en el catálogo: 69 "Apparent Power" con unidad kVAh; 3 y 68 ambos kVAh; 104 en Wh frente a 5 en kWh. Normalizar unidades antes de comparar.
- Los contadores de energía (60–62, 63–65, 68, 104, 5) son **acumulados**: el consumo de un periodo es `último − primero`, no un promedio.

## 5. Reglas sobre variables eléctricas

### 5.1 Hechos del motor [C]

- Una regla se evalúa sobre las lecturas de los componentes cuyo tipo coincide con `rule.component_id`, y en los handlers métricos **solo si `rule.unit_id == unit_id` de la lectura**. Si el medidor publica la magnitud con otro id, la regla nunca se evalúa y tampoco da error.
- Las reglas con `client_id = 0` son **globales**: modificarlas afecta a todos los clientes. Para ajustar un sitio o un equipo usar overrides, no editar la regla.
- `Rule.severity` es el tipo de evento ("Business Rule", "Anomaly", "diagnostic"), **no** la urgencia. La severidad real está en `get_rule_classifications` (el sitio gana al cliente) o en `get_alerts`.
- `trigger_detail` y `trigger_treshold` llegan como cadenas; convertir a número antes de comparar.
- `activate_on` > 0 crea el evento en `pending` y lo activa tras N minutos; `events_in_time` exige N lecturas en `timespan` minutos. Si la condición se despeja antes, el pending se **borra**, no pasa a closed.
- `groupable = true` reutiliza el evento abierto e incrementa `total`.

### 5.2 Handlers útiles para electricidad

| Handler | Formato | Uso eléctrico típico |
|---|---|---|
| `metric-by-component` | `trigger` = gt/lt/ge/le/eq/ne; `trigger_detail` umbral | Sobretensión, subtensión, sobrecorriente, frecuencia |
| `metric-by-range` | `trigger` = `nb;min;max` (dispara **fuera**) o `be;min;max` (dispara dentro) | Banda de tensión o de FP. `trigger_detail` no se usa |
| `metric-vs-schedule` | `trigger` = `op;true/false` | `gt;false` = sobre umbral **fuera de horario** (consumo en reposo); `gt;true` = solo en horario |
| `metric-vs-component` | `op1;componentId;componentCode;componentUnitId;op2;valor` | Corriente alta solo si otro equipo está en cierto estado (ej. generador) |
| `metric-by-period` | `comparador;deltaMin;HH:MM`, `timespan` negativo | Compara con el valor de un instante anterior más/menos `trigger_detail` |
| `abnormality` | Rango definido **solo** en overrides por componente | Rango normal propio de cada carga; sin override no evalúa |
| `capacity-component` | `trigger_detail` en % | Umbral = `Capacity × % / 100` sobre la capacidad del SiteComponent |
| `data-sync` / `disconnection` | códigos o `timeout` | Detección de silencio y reconexión |
| `metric-in-time` | — | **No implementado**; no recomendarlo |

Histéresis: con evento activo, `trigger_treshold` reemplaza a `trigger_detail`. Para `gt` el umbral de histéresis debe ser **menor** que el de disparo; para `lt`, **mayor**. Si falta, el evento puede abrirse y cerrarse con cada lectura que oscile.

### 5.3 Overrides [C]

Prioridad: componente > sitio > cliente. Dentro de un mismo nivel gana el id mayor. Cada override puede limitarse por `weekday` y por `start_at`/`end_at` en hora del sitio (admite cruce de medianoche). Dos trampas:

- Un override con `trigger_detail` ≤ 0 **no sustituye** el umbral de la regla (el motor exige > 0). Un umbral de 0 o negativo no se logra por override.
- En `metric-by-component`, si el SiteComponent tiene `Capacity > 0` el umbral se reinterpreta como **porcentaje de esa capacidad**. Verificar `Capacity` antes de dar un umbral por correcto.

### 5.4 Hallazgos observados (referencia: usar solo si la pregunta trata de reglas o de por qué una alerta no dispara; no auditar por iniciativa propia)

- Reglas eléctricas globales de tensión (tipo 88) con umbrales centinela `lt -30` y `gt 10000` en V: con sistemas de ≈120/208 V **no pueden dispararse**. Una regla activa no equivale a una regla útil.
- Reglas de frecuencia ligadas a unit_id 45 mientras los medidores revisados publican frecuencia con 301 o 181: nunca se evalúan.
- El nombre no siempre coincide con el unit_id: una regla "Ausencia de voltaje" evalúa unit_id 34 (corriente en catálogo) y otra "Phantom Shield L3" también evalúa 34 (Fase 1). **Siempre leer `rule.unit_id`, no el nombre.**
- En un sitio con medidor tipo 88 no había regla de pérdida de comunicación para ese tipo (las de `data-sync` eran de otros componentes). Verificar que cada tipo eléctrico tenga una.
- Una regla sin eventos en 30 días no prueba que funcione: puede estar desalineada por unit_id. Comparar siempre con los unit_id que el componente realmente emite.

## 6. Procedimientos con el MCP

Todas las llamadas necesitan `client_id`. En `get_component_rules`, `get_component_relations` y similares el parámetro `component_id` es el `site_component_id`, no el id del catálogo.

1. **Resolver nombres**: `find_assets(q, client_id)`; revisar `partial` antes de concluir que algo no existe.
2. **Inventario eléctrico del sitio**: `get_site_components` y filtrar por los component_id de §3. Si el total es exactamente 100, asumir truncamiento y cruzar con `get_site_relations`.
3. **¿Reporta?**: `get_current_state(scope_type="component", scope_id=...)` → `data_freshness` (live/stale/unknown) y `recent_events`. Si casi todo el sitio está stale a la vez, sospechar del enlace compartido (gateway, red) con `get_site_relations`, no de N medidores.
4. **Telemetría**: `get_measurements(site_component_id=...)` en modo agregado.
   - Tope de 5 000 filas: un medidor con 42 variables a 1 lectura/min se agota en ≈2 h; uno con 6 variables, en ≈14 h. **Si `window_truncated` es true, los agregados cubren solo el inicio de la ventana** (el `last_datetime` queda lejos del final).
   - Estrechar la ventana (`date_from`/`date_to` en `Y-m-d` o `Y-m-d H:i:s`) o usar `raw=true` con `after_id`. Rango máximo de `recent_days`: 30.
   - Para ver el estado actual usar `last_value`/`last_datetime_utc`, no `avg_value`.
5. **Reglas del componente**: `get_component_rules` y contrastar cada `rule.unit_id` con los unit_id emitidos en el paso 4. Marcar reglas huérfanas.
6. **Overrides**: `get_rule_custom_overrides(site_id, site_component_id, rule_id)`; revisar ventanas horarias y que `trigger_detail` > 0. Hay paginación (`has_more`).
7. **Alertas y severidad**: `get_alerts` para abiertas y severidad real; `get_events` para historia. Con solo `site_component_id`, `get_events` mira 1 día: pasar `range_id`.
8. **Retrospectiva**: `get_summary(scope=..., period=...)`. Las filas son ventanas móviles solapadas: **nunca sumarlas**. Un resultado vacío no significa "sin eventos"; leer `latest_available`.

## 7. Interpretación eléctrica [R]

Las comprobaciones se aplican **internamente** sobre las variables que el usuario preguntó. Se reportan solo si el resultado invalida el dato pedido (§0, advertencia puntual) o si el usuario pide un diagnóstico. No ampliar la consulta a otras variables para ejecutarlas.

| Comprobación | Esperado | Si no se cumple |
|---|---|---|
| V fase-fase ≈ √3 × V fase-neutro | 120 V → ≈208 V (observado 121,5 → 211,8) | Mal conexionado, ids mal asignados o sistema distinto |
| S² = P² + Q² | Coherente | Unidad o escala errónea |
| FP = P / S | Coincide con el reportado | Lecturas de instantes distintos o unit_id mal interpretado |
| Corriente > 0 con tensión ≈ 0 en la misma fase | No es físico | Tensión no cableada o TC en fase equivocada |
| Q grande y negativo con FP bajo y corriente alta | Raro en cargas comunes | Posible TC invertido o desfasado |
| Frecuencia | Cercana a 60 Hz en Colombia | Fuera de ±1 Hz suele indicar planta o problema de red |
| Corrientes de las tres fases | Similares en cargas trifásicas | Desbalance (calcular (máx−media)/media) |

Caso real de la comprobación 4: un medidor trifásico mostró 125 V solo en L1 (L2-N ≈0,6 V y L3-N ≈0,3 V) con ≈23 A circulando en L2 y L3. La potencia activa de L2 y L3 quedó en ≈0,01 kW. Es decir, **el consumo de dos fases no se estaba midiendo**; la potencia total reportada estaba subestimada. La causa más probable es de cableado o configuración, no de plataforma.

Tolerancias típicas de tensión: ±5 a ±10 % sobre la nominal; FP de referencia ≥0,90; THD de corriente según carga y norma. Son puntos de partida, no configuración de Reveal. Confirmar con el cliente y con RETIE/NTC aplicable.

## 8. Diagnóstico por síntoma

Usar solo cuando el usuario describe un síntoma o pide una causa. Seguir el orden y detenerse en la primera causa confirmada.

| Síntoma | Revisar en orden |
|---|---|
| No llegan datos | freshness del componente → ¿todo el sitio stale? → relaciones `sendsDataTo` hacia el gateway → eventos de desconexión |
| Valores planos o en cero | ¿unit_id correcto? → ¿medidor sin carga? → escala/unidad → lectura de un contador acumulado |
| Magnitud absurda (negativa, ×1000) | signo/convención, unidad (W vs kW, Wh vs kWh), unit_id mal asignado, TC invertido |
| Alerta que no se dispara | `status` de la regla → coincidencia de `unit_id` → override con ventana horaria → `activate_on`/`events_in_time` → `skip_outside_schedule`/`skip_on_holidays` → `discon_parent_validation` o `parent_validation` (suprimen si hay un evento padre) → umbral centinela → `Capacity` |
| Alerta que no cierra | reconexión manual (override con `reconnection_method` > 0) → histéresis mal orientada → condición realmente vigente |
| Demasiadas alertas | `groupable` en falso, sin histéresis, `activate_on` = 0 con lecturas ruidosas |
| Severidad inesperada | `get_rule_classifications` con `rule_id` y `site_id`; leer `effective` |

## 9. Límites conocidos de las herramientas [C]

- `list_variables` y `get_site_components` devolvieron exactamente 100 elementos, sin paginación disponible.
- `get_measurements` agregado escanea como máximo 5 000 filas.
- `list_brick_*` y `get_site_relations` solo muestran componentes con relaciones.
- Los datos (descripciones, reglas, relaciones) vienen en español; los enums, en inglés.
- `logout` es irreversible en la sesión: no usarlo salvo petición expresa.
- El MCP es de **lectura**: no crea ni modifica reglas, overrides ni variables. Proponer el cambio y quién debe aplicarlo.

## 10. Cómo responder

Aplica siempre la regla de alcance de §0.

1. Responder primero la pregunta, en la primera línea, con valor, unidad y `datetime_utc`.
2. Mencionar cliente, sitio, componente y ventana solo en una línea corta; indicar truncamiento únicamente si afecta el dato entregado.
3. Marcar el nivel de confianza ([P], [?]) solo en los datos que lo requieren y que el usuario pidió. Si un unit_id pedido no está confirmado, decirlo en lugar de nombrarlo.
4. Nada de secciones, tablas ni listas si la respuesta cabe en una o dos frases.
5. No cerrar con acciones ni recomendaciones, salvo que el usuario las pida o aplique la advertencia puntual de §0.

Ejemplo correcto. Pregunta: "¿Cuál es la tensión L1 del medidor X ahora?"
Respuesta: "125,1 V (unit_id 37), última lectura 2026-10-09 14:02 UTC."

Ejemplo incorrecto: la misma respuesta seguida de corriente, potencia, FP, frecuencia, estado de reglas o análisis de desbalance que nadie pidió.

## 11. Pendientes para endurecer este skill

- Obtener el catálogo completo de Variables (ids >104) y convertir la tabla §4.2 de [P] a [C].
- Confirmar qué unit_id usa cada modelo de medidor (ME337, PZEM, CVM-A1500, UPS SNMP) para la misma magnitud.
- Validar con el equipo de plataforma el efecto de `reconnection_method="manual"` en reglas métricas.
- Verificar qué tipos eléctricos carecen de regla de pérdida de comunicación en toda la plataforma.
