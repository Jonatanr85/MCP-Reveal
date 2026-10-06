---
name: mcp-reveal
description: Procedimiento para consultar datos del servidor MCP Reveal (inventario, estado, alertas, eventos, mediciones, reglas y topología BRICS) resolviendo primero los IDs y validando cada respuesta. Úsalo cuando el usuario pida datos de la plataforma Reveal o pida depurar una respuesta vacía o inconsistente de sus herramientas.
---

# Consultar el MCP Reveal

Procedimiento de 5 pasos. Sigue el orden. Entrada: una solicitud del usuario sobre datos de Reveal. Salida: una respuesta con hallazgo, evidencia y limitaciones, basada solo en llamadas hechas en esta sesión.

## Paso 0. Verificar conexión

1. Comprueba que existan las herramientas del Anexo A en tu lista (identifícalas por sufijo, ej. `get_events`; el prefijo varía por cliente).
2. Llama `auth_status`. Si `authenticated` no es `true`, detente.
3. Si falta alguna herramienta o no hay sesión, informa al usuario que el servidor MCP no está conectado y cita la configuración del Anexo B. No inventes datos.

## Paso 1. Clasificar la solicitud

| Si el usuario pide... | Herramientas principales |
|---|---|
| Qué equipos hay en un sitio | `get_site_components` (+ `list_type_components` para interpretar tipos) |
| Cómo está algo ahora | `get_current_state`, luego `get_alerts` para el detalle |
| Historial de fallas o eventos | `get_events` con ventana explícita |
| Valores de variables o telemetría | `get_measurements` con ventana explícita (+ `list_variables`) |
| Por qué se disparó una alerta o su severidad | `get_component_rules` / `get_site_rules`, `get_rule_classifications` |
| Cómo se conectan los equipos | `get_site_relations`, `get_component_relations`, o `list_brick_edges` para todo el cliente |
| Reportes pre-agregados del motor de reglas | `get_summary` |
| Personas con acceso | `list_client_users` (solo si lo piden) |

## Paso 2. Resolver identificadores

1. Si el usuario nombró un recurso en texto libre, llama `find_assets` (revisa `partial` antes de concluir que no existe).
2. Si no, usa `clients` y haz coincidencia parcial sin distinguir mayúsculas; luego `list_sites` y `get_site_components`.
3. Si hay 0 o más de 1 coincidencia razonable, muestra las opciones y pregunta. No asumas.
4. `client_id` es siempre el que devuelve `clients()`. Un ID dado por el usuario que no aparezca ahí no se usa.
5. En `get_component_rules` y `get_component_relations`, el parámetro `component_id` es un `site_component_id` (el `id` de `get_site_components`), no el tipo de dispositivo.

## Paso 3. Ejecutar la consulta

- Pasa siempre una ventana explícita en `get_events`, `get_alerts` y `get_measurements` (`range_id`, `recent_days` o `date_from` + `date_to`). Filtrar solo por `site_component_id` aplica una ventana implícita de 1 día.
- Pagina mientras `has_more` sea verdadero o `total_records` supere lo recibido.
- Usa `get_current_state` solo como triage: su `alerts[]` es un top-N. Si `open_alerts` es mayor, confirma con `get_alerts`.

## Paso 4. Validar el resultado

Antes de responder, revisa:
- `has_more` / `total_records`: ¿cubres todo? Si no, dilo.
- `partial` (en `find_assets`) y `window.empty_result_note` (en eventos y alertas): un vacío puede ser una falla parcial o una ventana corta.
- `window_truncated` (en `get_measurements`): si es verdadero, el agregado no cubre la ventana.
- 404 `ENTITY_NOT_FOUND` significa que no existe; 200 con lista vacía significa que existe sin coincidencias. No los trates igual.
- `get_summary`: nunca sumes filas entre sí; `summaries[]` vacío significa que el pipeline no corrió.
- Severidad real: viene de `get_rule_classifications`, no del campo `severity` de `Rule` ni de `get_events`.

## Paso 5. Responder

Formato: (1) hallazgo en una frase, (2) evidencia (cifras y fechas devueltas por las herramientas), (3) limitaciones (ventana usada, datos parciales, vacíos). No muestres IDs internos salvo que el usuario los pida. Si una llamada falló, repórtalo.

## Criterios de éxito

- Todo dato citado proviene de una llamada de esta sesión.
- Se resolvieron los IDs antes de consultar (Paso 2).
- La ventana de tiempo fue explícita.
- Las limitaciones del Paso 4 quedaron declaradas.

## Casos de prueba

1. "Qué equipos tiene el sitio X del cliente Y": debe llamar `clients` (o `find_assets`), `list_sites`, `get_site_components`, y responder con totales por tipo.
2. "Qué alertas críticas hay abiertas en el sitio X": debe llamar `get_current_state` y luego `get_alerts` con `statuses=active`, y comparar el conteo con `open_alerts`.
3. "A qué equipos alimenta el equipo Z": debe llamar `find_assets`, luego `get_component_relations`, y reportar las aristas con su dirección.
4. Sin conexión al MCP: debe detenerse en el Paso 0 e informar, sin devolver datos.

## Anexo B. Configuración requerida del cliente de IA

- URL del servidor MCP: `https://mcp.gssanalytix.com/mcp`
- Transporte esperado: HTTP streamable (remoto), pendiente de confirmar. Si el cliente solo admite SSE o stdio, usa un puente como `mcp-remote`.
- Autenticación esperada: OAuth. Sin sesión, el servidor responde 401.

## Anexo C. Modelo de datos

```
Cliente (client_id)
  └─ Sitio (site_id)
       └─ Componente instalado (site_component_id)
            ├─ Reglas (rule_id)
            ├─ Eventos / alertas
            ├─ Mediciones (variables / unidades)
            └─ Relaciones BRICS (aristas hacia otros componentes)
```

Todas las herramientas requieren `client_id`, excepto `clients`, `auth_status`, `logout`, `list_type_components` y `list_variables`.

## Anexo A. Herramientas (23)

### Autenticación
- `auth_status`: devuelve `authenticated`, `user`, `reason`. Revisa `authenticated` directamente; la ausencia de `reason="logged_out"` no implica sesión válida.
- `logout`: revoca el token de la sesión y es irreversible. Solo si el usuario lo pide.

### Catálogos y resolución de IDs
- `clients`: lista de clientes accesibles. Sin paginación.
- `list_sites`: sitios de un cliente. `country` es un ID legado de calendario; usa `country_iso` / `country_name`.
- `get_site_components`: dispositivos instalados en un sitio. Devuelve `id` (= `site_component_id`), `description`, `component_id`, `component_name`.
- `list_type_components`: catálogo global de tipos de dispositivo. Interpreta `component_id`.
- `list_variables`: catálogo global de variables/unidades. `unit_id` en reglas y eventos referencia este catálogo.
- `find_assets(client_id, q, type?, limit?)`: resuelve texto libre (`q`, mínimo 2 caracteres) a IDs con su cadena de padres. `type`: client | site | component; si se omite busca en los tres. `limit` 1–50 (por defecto 10). `variable` no está soportado. `component` significa equipo instalado, no el catálogo de tipos. Revisa el campo `partial` antes de concluir que algo no existe.

### Estado operativo
- `get_current_state`: snapshot de costo fijo. `scope_type` = client | site | component | variable (variable requiere `anchor_id` y `anchor_type`). `alerts[]` es un top-N, no la lista completa; compáralo con `open_alerts`.
- `get_alerts`: alertas (subconjunto de eventos). Por defecto solo `active`. Si filtras solo por `site_component_id`, la ventana implícita es de 1 día; pasa `range_id` o `date_from` + `date_to`. Revisa `window.empty_result_note` antes de concluir que no hay historial.
- `get_events`: historial completo de notificaciones. `is_alert=true` marca las que son alerta. Pagina con `start` + `limit` o con `min_id`, sin mezclar. Los eventos `pending` pueden salir primero; ordena localmente si necesitas orden cronológico.
- `get_summary`: reportes pre-agregados del motor de reglas, sobre ventanas móviles que se solapan. Nunca sumes filas entre sí. `summaries[]` vacío significa que el pipeline no corrió, no que no hubo eventos. `period` define el tipo de reporte, no un filtro de fecha.
- `get_measurements`: telemetría. Modo componente (`site_component_id`, recomendado) o modo feed (`site_id` / `site_component_ids` con cursor `start`, que es un ID y no un offset). Ventana obligatoria: `recent_days` (máx. 30) o `date_from` + `date_to`. Con `raw=false` se ignora `limit`; si `window_truncated=true`, el agregado no cubre toda la ventana.

### Reglas
- `get_site_rules`: reglas de un sitio. No trae conteo de activas; filtra `rules[]` por `status`.
- `get_component_rules`: reglas de un dispositivo (usa `site_component_id`).
- `get_rule_custom_overrides`: overrides de parámetros por sitio, componente o cliente (gana el más específico). `reconnection_method` es entero aquí, no el texto que trae `Rule`. 404 `ENTITY_NOT_FOUND` significa que no existe; 200 con lista vacía significa que existe sin coincidencias.
- `get_rule_classifications`: fuente de verdad de la severidad de alertas. El campo `severity` de `Rule` y de `get_events` no es la severidad real. Una fila con `site_id` sobrescribe la de nivel cliente solo para ese sitio. Con `rule_id`, devuelve `effective`.

### Grafo BRICS
Grafo dirigido: vértices = componentes instalados, aristas = relaciones (ej. `powers`, `sendsDataTo`, `suppliesFuelTo`).
- `get_site_relations`: topología de un sitio. Usa `edges[]`; los campos `outgoing_edges`, `incoming_edges` y los `*_count` están deprecados y vacíos aquí.
- `get_component_relations`: conexiones de un componente. Aquí `outgoing_edges` / `incoming_edges` y sus conteos sí son los campos vivos.
- `get_component_relations_bulk`: varios componentes en una llamada. `direction`: out | in | both (por defecto `both`). Un receptor puro devuelve 0 filas con `out`; usa `both` o `in` (verificado en vivo: con `out` devolvió `total: 0` y sin `direction` devolvió la relación entrante). Cada relación trae `from_vertex_id`, `to_vertex_id`, `label` (ej. `sendsDataTo`), `relation_name_from`, `relation_name_to` y `component_from` / `component_to`. El campo `direction` (`incoming`) solo viene poblado en modo `both`. IDs separados por coma, enteros positivos.
- `list_brick_vertices(client_id, label?, source_type?, limit?, start?)`: vértices de todo el cliente. `label` filtra por descripción (coincidencia parcial) y `source_type` por tipo de componente. Solo incluye componentes con al menos una relación; no es el inventario completo. `limit` máx. 1000. Pagina con `start` + `limit` hasta `has_more=false`.
- `list_brick_edges`: aristas de todo el cliente, mismo patrón de paginación. Úsala para auditar el grafo completo sin iterar sitio por sitio.

### Directorio de acceso
- `list_client_users`: personas con cuenta y su rol. Es control de acceso, no datos operativos. Es información personal: solo si el usuario la pide. Sin paginación; filtra localmente.

