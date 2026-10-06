---
name: mcp-reveal
description: Referencia técnica del servidor MCP Reveal (plataforma de monitoreo por cliente, sitio y componente). Úsala cuando el usuario pregunte qué herramientas ofrece el MCP Reveal, cómo encadenarlas, qué es el grafo BRICS, o cuando una respuesta de estas herramientas sea vacía, confusa o inconsistente y haya que depurarla.
---

# MCP Reveal — referencia de herramientas y grafo BRICS

## 0. Prerrequisitos (leer primero)

Este archivo es documentación. No da acceso al servidor. Para usar las herramientas, el cliente de IA debe tener el servidor MCP Reveal configurado y autenticado:

- URL del servidor MCP: `https://mcp.gssanalytix.com/mcp`
- Tipo de transporte: HTTP streamable (remoto). Si tu cliente solo admite SSE o stdio, usa un puente como `mcp-remote`.
- Autenticación: OAuth. Sin sesión autenticada, el servidor responde 401. El cliente debe completar el flujo de inicio de sesión en el navegador la primera vez.

Reglas para la IA que lee este archivo:

1. Verifica que las herramientas de la sección 3 existan en tu lista de herramientas. Si no existen, informa al usuario que el servidor MCP no está conectado. No inventes datos ni respuestas.
2. Los nombres de herramienta pueden venir con prefijo según el cliente (ej. `mcp__MCP_Reveal__get_events`, `reveal_get_events`). Identifica cada herramienta por su sufijo (`get_events`).
3. El esquema real de cada herramienta en tu entorno tiene prioridad sobre este documento. Si un parámetro difiere, usa el del esquema.
4. Los recursos `reveal://data-model` y `reveal://rule-handlers` pueden no estar disponibles en tu cliente. No dependas de ellos.

## 1. Modelo de datos

```
Cliente (client_id)
  └─ Sitio (site_id)
       └─ Componente instalado (site_component_id)
            ├─ Reglas (rule_id)
            ├─ Eventos / alertas (activaciones de reglas)
            ├─ Mediciones (variables / unidades)
            └─ Relaciones BRICS (aristas hacia otros componentes)
```

- `client_id` es siempre el numérico que devuelve `clients()`. Si el usuario da un ID que no aparece ahí, no lo uses.
- Todas las herramientas requieren `client_id`, excepto: `clients`, `auth_status`, `logout`, `list_type_components`, `list_variables`.
- `component_id` tiene dos significados. En `get_site_components` es el tipo de dispositivo (catálogo global). En `get_component_rules` y `get_component_relations` el parámetro llamado `component_id` es en realidad un `site_component_id` (el `id` devuelto por `get_site_components`).

## 2. Flujo típico

```
clients
 → list_sites
    → get_site_components      (inventario)
    → get_site_relations       (topología del sitio)
    → get_current_state        (triage rápido)
       → get_alerts / get_events   (detalle)
          → get_component_rules    (qué regla se disparó)
             → get_rule_classifications  (severidad real)
```

`find_assets` permite saltar los primeros pasos a partir de un nombre en texto libre.

## 3. Herramientas (23)

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

## 4. Reglas de rigor

- Todo dato debe provenir de llamadas hechas en esta sesión. Nunca de memoria ni de ejemplos.
- Si una llamada falla o devuelve vacío, repórtalo explícitamente. No lo rellenes.
- Pagina hasta agotar resultados (`has_more`, `total_records`) o advierte que los datos son parciales.
- Distingue "no existe" (404) de "existe sin datos" (200 vacío).
- No muestres IDs internos al usuario final salvo que los pida.
