# SKILL: Informe gerencial del estado CCTV (MCP Reveal)

Versión portable. Ejecutable por cualquier IA/agente que pueda invocar herramientas del MCP Reveal. No depende de funciones propias de un proveedor.

## 1. Propósito y activación

Consultar en vivo el estado actual y detallado de la infraestructura CCTV de un cliente en Reveal y entregar un informe gerencial sobre:

- Grabadores (NVR / DVR / XVR)
- Discos
- Cámaras análogas
- Cámaras IP

**Activar cuando:** el usuario escriba `/CCTV`, o pregunte por el estado de cámaras, grabadores, DVR, NVR, discos de video, conectividad CCTV o pida un informe de videovigilancia.

## 2. Requisitos

Herramientas del MCP Reveal (nombres exactos):
`clients`, `list_sites`, `get_site_components`, `get_current_state`, `get_events`, `get_alerts`, `list_type_components` (opcional, para validar el catálogo), `find_assets` (opcional, para resolver nombres).

Si alguna no está disponible, detente e informa cuál falta. No sustituyas datos con memoria, suposiciones ni archivos externos.

## 3. Reglas generales

1. Todo dato del informe proviene de llamadas al MCP en esta ejecución. Nunca de memoria, ejemplos previos ni Excel/correos.
2. Si algo es ambiguo o falta información, pregunta antes de continuar. No asumas.
3. Si una llamada falla o devuelve vacío, repórtalo explícitamente. No lo omitas ni lo rellenes.
4. Responde de forma breve, directa y sin halagos. Si el usuario afirma algo que contradice los datos de Reveal, corrígelo con evidencia.
5. `client_id` es el numérico que devuelve `clients()`. No reutilices IDs de otras plataformas ni de ejecuciones anteriores.

## 4. Flujo

### Paso 1 — Cliente
Si el usuario no indicó cliente, pregunta: "¿Para qué cliente genero el informe CCTV?". Con el nombre, llama `clients()` y busca coincidencia parcial sin distinguir mayúsculas en `name`. Si hay 0 o más de 1 coincidencia, muestra las opciones y pide confirmación.

### Paso 2 — Sitios
Llama `list_sites(client_id)`.
- Si el usuario nombró un sitio, filtra por coincidencia parcial.
- Si no, usa todos. Si hay más de 8 sitios, pregunta si quiere todos o un subconjunto.

### Paso 3 — Inventario CCTV
Por sitio: `get_site_components(client_id, site_id)`. Filtra por `component_id`:

| Categoría | component_id | Nombre Reveal |
|---|---|---|
| Grabadores | 103 | NVR |
| Grabadores | 15 | DVR inhouse |
| Grabadores | 146 | DVR IP (GatewayNVR) |
| Grabadores | 193 | XVR |
| Discos | 130 | Disco Duro |
| Discos | 10 | Inhouse Video Disco Duro |
| Cámaras análogas | 16 | Analog Camera |
| Cámaras IP | 69 | IP Camera |

Ignora todo lo demás (medidores, switches, PLC, etc.). Excluye del informe los sitios sin componentes CCTV. Si un `component_id` con `system: CCTV` no está en la tabla, valídalo con `list_type_components()` y repórtalo como "otro CCTV" sin incluirlo en métricas.

`id` del componente = `site_component_id` (se usa en eventos). `description` suele traer el DVR padre en el texto (ej. "DVR - Herramenteria Canal 3: ..."). Úsalo para agrupar cámaras por grabador.

### Paso 4 — Triage rápido por sitio
`get_current_state(client_id, scope_type="site", scope_id=site_id, recent_days=1)`.
- Guarda `generated_at` (hora oficial del informe), `open_alerts` y `data_freshness[]` (`live` / `stale` por componente).
- `alerts[]` es un top-N por severidad, NO exhaustivo. Si `open_alerts` > longitud de `alerts[]`, es obligatorio el Paso 5.

### Paso 5 — Fallas activas (fuente de verdad)
`get_alerts(client_id, site_id=site_id, statuses="active", limit=1000)`.
Complementa con `get_events(client_id, site_id=site_id, statuses="active", range_id="30d", limit=1000)` si necesitas eventos no clasificados como alerta.

Pagina con `start` mientras `has_more` sea verdadero o `total_records` supere lo recibido. No reportes con datos parciales sin advertirlo.

Campos útiles por evento: `site_component_id`, `equipo`, `falla` (nombre de la regla), `type`, `domain`, `status`, `severity`, `activated_at`, `total` (activaciones acumuladas), `recommendation`.

Conserva solo eventos cuyo `site_component_id` esté en el inventario CCTV del Paso 3.

### Paso 6 — Historial para inestabilidad (opcional pero recomendado)
`get_events(client_id, site_id=site_id, statuses="active,closed", range_id="7d", limit=1000)`. Cuenta activaciones por componente. Más de 3 en 7 días = "inestable".

Trampa conocida: al filtrar solo por `site_component_id`, Reveal aplica una ventana implícita de 1 día. Pasa siempre `range_id` y revisa `window.empty_result_note` antes de concluir que no hay historial.

## 5. Clasificación de estado por componente

Vocabulario de conectividad definido por el negocio:
- **Falla:** `disconnected`, `unreachable`, `loginFail`
- **Recuperación:** `connected`, `reachable`, `normalLogin`, `normalConnection`

Busca estos términos (inglés y su equivalente en español: desconectado, inalcanzable, fallo de login, etc.) en `type`, `domain`, `parameter`, `falla` y `recommendation` (ej. `event_code=disconnected`). Los nombres literales pueden no aparecer según la configuración de reglas del cliente; si no aparecen, dilo en el informe.

Reglas observadas en producción (interprétalas por el texto de `falla`, no solo por `type`):

| Texto en `falla` | Interpretación |
|---|---|
| "…Desconectada" (cámara análoga / IP) | OFFLINE |
| "…sin grabación" (cámara) | OFFLINE (no graba) |
| "Anomalía detectada - Disco Duro" / "Canal de Comunicación Disco Duro" | Disco con FALLA |
| "Detección de Movimiento…" aunque `type=disconnection` | NO contar como offline. Marcar "A VERIFICAR" |

Estados:
- **OFFLINE:** evento de falla con `status="active"`.
- **ONLINE:** sin evento de falla activo y `data_freshness=live`.
- **ADVERTENCIA:** sin falla activa pero `data_freshness=stale`, o inestable (>3 activaciones/7 días), o "A VERIFICAR".
- Un evento `closed` o inexistente no es OFFLINE. El estado nunca se infiere del nombre del canal ni de otra fuente.

**Heurística de grabador caído (hipótesis, no hecho):** si ≥3 cámaras del mismo grabador (por `description`) tienen la misma falla activada en una ventana de ≤60 minutos, repórtalo como "Posible falla del grabador o su red/almacenamiento — requiere verificación". No cambies el estado del grabador a OFFLINE si Reveal no tiene un evento propio sobre él. Umbral configurable; por defecto 3.

**Antigüedad:** calcula días activo desde `activated_at`. Falla > 30 días = "crónica"; destácala aparte.

## 6. Métricas

Por sitio y categoría: total, online, offline, advertencia, % disponibilidad = (total − offline) / total. Cada componente cuenta en un solo estado. Global = suma de las cuatro categorías. Calcula con código o de forma verificable; no estimes de cabeza. Los "A VERIFICAR" no restan disponibilidad, pero se muestran aparte.

## 7. Salida

Por defecto: **un archivo HTML autocontenido** (CSS y JS en línea, sin recursos externos, responsive, legible en claro y oscuro). Si el usuario pide otro formato (PDF, Word, Excel, texto en chat), entrégalo en ese formato con la misma estructura.

Estructura obligatoria:
1. **Encabezado:** cliente, sitios, fecha/hora (`generated_at`), total de componentes evaluados.
2. **Resumen ejecutivo:** 4 KPIs (% disponibilidad global, componentes OFFLINE, componentes en advertencia/a verificar, fallas crónicas) y el hallazgo principal en una frase.
3. **Estado por categoría:** tabla por sitio (total / online / offline / advertencia / %).
4. **Hallazgos críticos:** ordenados por severidad; incluye hipótesis de grabador caído (rotuladas como hipótesis), fallas crónicas (días activo y `total`), discos con anomalías.
5. **Detalle por sitio:** tabla completa de componentes con estado, falla, desde cuándo.
6. **Contraste con fuente externa:** solo si el usuario aportó otra fuente (Excel, correo). Tabla lado a lado y % de precisión de esa fuente frente a Reveal.
7. **Pie:** metodología en una línea (criterio de evento activo, fecha del snapshot).

No mostrar en el informe: `client_id`, `site_component_id`, `rule_id` ni nombres de herramientas. No usar color como único indicador (acompañar con texto). Tono ejecutivo, sin relleno.

## 8. Lista de verificación antes de entregar

- [ ] Todos los números provienen de llamadas de esta ejecución.
- [ ] Ningún conteo se hizo "a ojo": verificado con código o recuento explícito.
- [ ] Se paginó hasta agotar resultados o se advirtió la truncación.
- [ ] Los eventos de movimiento/otros no-conectividad no inflan OFFLINE.
- [ ] Las hipótesis están rotuladas como hipótesis.
- [ ] Vacíos y errores de llamadas están declarados.
- [ ] Los totales por categoría suman el total global.

## 9. Ejemplo de invocación

Usuario: `/CCTV`
IA: "¿Para qué cliente genero el informe CCTV?"
Usuario: "Ladrillera Santafé, Soacha"
IA: ejecuta Pasos 1–7 y entrega el HTML.
