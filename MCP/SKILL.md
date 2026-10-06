# MCP_REVEAL: Referencia de herramientas y grafos — MCP Reveal

Skill de referencia técnica, portable a cualquier IA/agente con acceso al MCP Reveal.
No asume ningún cliente, sistema o proceso de negocio: cubre únicamente el contrato de
las herramientas, cómo se relacionan entre sí, y el modelo de grafo que exponen.

**Activar cuando:** el usuario pregunte qué herramientas tiene disponibles el MCP Reveal,
cómo se relacionan entre sí, qué es el grafo BRICS, cómo encadenar llamadas, o pida ayuda
para depurar/entender una respuesta de alguna de estas herramientas.

**Nota de alcance:** existe un servidor MCP distinto (también llamado "Reveal" en algunos
entornos) con herramientas de nombre similar (`get_node`, `get_neighbors`, `query_graph`,
`shortest_path`, `list_prs`, etc.) que opera sobre un grafo de conocimiento de código/PRs
de un repositorio — no tiene relación con lo descrito aquí. Si el usuario menciona nodos,
PRs o repositorios, confirma primero cuál de los dos servidores está usando antes de aplicar
esta skill.

## 1. Modelo mental

El MCP Reveal expone datos de una plataforma de monitoreo jerárquica:

```
Cliente (client_id)
  └─ Sitio (site_id)
       └─ Componente instalado (site_component_id)
            ├─ Reglas de notificación (rule_id)
            ├─ Eventos/alertas (activaciones de esas reglas)
            ├─ Mediciones (lecturas de variables/unidades)
            └─ Relaciones BRICS (aristas hacia otros componentes)
```

Todas las herramientas requieren `client_id` salvo `clients()`, `auth_status()`,
`logout()` y `list_type_components()`/`list_variables()` (catálogos globales, sin scope
de cliente).

`client_id` es SIEMPRE el numérico devuelto por `clients()`. No debe confundirse con
ningún identificador de cliente de otro sistema o plataforma — si el usuario da un ID
que no aparece en `clients()`, no lo asumas válido; resuélvelo primero.

## 2. Autenticación

| Herramienta | Uso |
|---|---|
| `auth_status()` | Devuelve `{authenticated, user, reason}`. `reason="logged_out"` solo aparece si esta sesión llamó `logout()`; su ausencia NO implica `authenticated=true` — revisa ese campo directamente. |
| `logout()` | Revoca el token de la sesión. Irreversible dentro de la sesión — no hay forma de reautenticar sin que el conector se reconecte. Úsala solo si el usuario lo pide explícitamente. |

## 3. Catálogos y resolución de identificadores

Estas herramientas no dependen de conocer IDs de antemano — son el punto de entrada.

| Herramienta | Qué resuelve | Notas |
|---|---|---|
| `clients()` | Lista de organizaciones cliente accesibles al token | Sin paginación (catálogo pequeño). Un token de un solo cliente solo ve ese uno. |
| `list_sites(client_id)` | Sitios (edificios/instalaciones) de un cliente | `sites[].country` es un ID numérico legado de calendario de festivos, NO un código ISO — usa `country_iso`/`country_name`. |
| `get_site_components(client_id, site_id)` | Todos los dispositivos físicos instalados en un sitio | Devuelve `{id (=site_component_id), description, component_id, component_name}`. `component_id` aquí es la referencia al catálogo global de tipos — no confundir con `site_component_id`. |
| `list_type_components()` | Catálogo global de ~150+ tipos de dispositivo (referencia, no instalados) | Úsalo para interpretar `component_id` de `get_site_components`. |
| `list_variables()` | Catálogo global de variables de medición/unidades | `unit_id` en `Rule`/`Event` referencia este catálogo (ej. unit_id=3 → "Celsius"). |
| `find_assets(client_id, q, type?)` | Resuelve texto libre ("router bogota") a IDs exactos con su cadena de padres | Sustituye el encadenado manual `clients → list_sites → get_site_components`. `type` puede ser `client`/`site`/`component` (omitir para buscar en los tres); `variable` NO está soportado. Revisa el campo `partial` antes de concluir que algo no existe — una falla parcial en un tipo puede lucir igual que un resultado vacío real. |

## 4. Estado operativo (qué está pasando ahora / qué pasó)

| Herramienta | Para qué | Detalles clave |
|---|---|---|
| `get_current_state(client_id, scope_type, scope_id?, anchor_id?, anchor_type?, recent_days?)` | Snapshot de costo fijo: alertas abiertas, top-N por severidad, conteo de eventos recientes y (salvo `scope_type="client"`) frescura de datos por componente | `scope_type`: `client`\|`site`\|`component`\|`variable`. Para `variable` se requieren `anchor_id`+`anchor_type` (`site`\|`component`). Es un triage rápido, NO exhaustivo — `alerts[]` es top-N, no la lista completa. |
| `get_alerts(client_id, ...)` | Notificaciones clasificadas como alerta (subconjunto estricto de `get_events`) | `statuses` por defecto `"active"` (abiertas). Filtrar solo por `site_component_id` activa una ventana implícita de 1 día — usa `range_id` para ampliarla y revisa `window.source`/`window.empty_result_note` antes de concluir que no hay historial. |
| `get_events(client_id, ...)` | Historial completo de notificaciones (tabla completa, no solo alertas) | `is_alert=true` marca las que también son alerta. Paginación por `start`+`limit` (offset) o `min_id` (cursor) — no mezclar ambos. Orden por defecto pone `status="pending"` primero (su `activated_at` es null y los NULL ordenan antes en DESC) — ordena localmente si necesitas orden cronológico real. |
| `get_summary(client_id, ...)` | Reportes pre-agregados del motor de reglas (rules-engine output), NO telemetría cruda ni estado en vivo | Cada fila es una corrida sobre una ventana ROLLING, no un período de calendario — nunca sumar entre filas, se solapan. `summaries[]` vacío significa que el pipeline no corrió para ese scope (revisar `latest_available`/`empty_result_note`), no "sin eventos". `period` define el TIPO de reporte, no un filtro de fecha (usar `date`/`date_from`+`date_to` para eso). `trend`/`advanced_stats`/`correlation` siempre usan ventana fija de 90 días sin importar `period`. |
| `get_measurements(client_id, ...)` | Telemetría/lecturas del datafeed (`datalog`), agregada por defecto | Dos modos: componente único (`site_component_id`, recomendado, soporta cursor `after_id`) o feed (`site_id`/`site_component_ids`, requiere cursor `start` — que es un ID, NO un offset). Ventana obligatoria: `recent_days` (máx. 30) o `date_from`+`date_to`. `limit` se ignora si `raw=False` (agregado escanea toda la ventana internamente); si `window_truncated=True`, el agregado no cubre toda la ventana pedida — hay que acotarla o usar `raw=True`. |

## 5. Reglas de notificación

| Herramienta | Alcance | Notas |
|---|---|---|
| `get_site_rules(client_id, site_id)` | Reglas aplicadas a todos los dispositivos de un sitio | No hay campo de conteo de activas — filtrar `rules[]` por `status` localmente. 17 tipos de handler (ver `reveal://rule-handlers`). |
| `get_component_rules(client_id, component_id)` | Reglas de un único dispositivo | Mismo shape que `get_site_rules`. `component_id` aquí es un `site_component_id`, pese al nombre del parámetro. |
| `get_rule_custom_overrides(client_id, ...)` | Overrides de parámetros de regla por sitio/componente/cliente (el más específico gana) | `reconnection_method` es un entero (0=automático, >0=manual) — distinto del enum de texto que trae `Rule`, no compararlos directamente. `site_component_id=0` REQUIERE `site_id` (filtra localmente filas con scope 0). Un `site_id`/`site_component_id` inexistente → 404 (`ENTITY_NOT_FOUND`); un `rule_id` sin coincidencias → 200 con `triggers[]` vacío — no confundir "no encontrado" con "sin resultados". |
| `get_rule_classifications(client_id, rule_id?, site_id?)` | Mapeo regla → clasificación de alerta y severidad real | Es la fuente de verdad de severidad (el campo `severity` en `Rule`/`get_events` es un nombre engañoso). Una fila con `site_id` sobrescribe la fila de nivel cliente (`site_id=null`) solo para ese sitio. El filtrado por `rule_id`/`site_id` en upstream se ignora — esta herramienta trae la lista completa del cliente y filtra localmente. Pasar `rule_id` también devuelve `effective`, la fila aplicable resuelta. |

## 6. El grafo BRICS (topología de conexiones físicas/operativas)

BRICS modela cómo los componentes instalados se conectan entre sí: alimentación
eléctrica, flujo de datos, suministro, etc. (ej. tipos de arista: `powers`,
`sendsDataTo`, `suppliesFuelTo`). Es un grafo dirigido: **vértices** = componentes
instalados (`site_component_id`), **aristas** = relaciones entre ellos.

| Herramienta | Alcance | Notas |
|---|---|---|
| `list_brick_vertices(client_id, label?, source_type?, limit?, start?)` | Vértices accesibles al cliente, a nivel de TODO el cliente (no un sitio) | Solo incluye componentes que participan en ≥1 relación — NO es el inventario completo (para eso, `get_site_components`). Ordenado por `id` ascendente; paginar con `start`+`limit` hasta `has_more=false`. |
| `list_brick_edges(client_id, limit?, start?)` | Todas las aristas accesibles al cliente, a nivel de TODO el cliente | Evita iterar `get_site_relations`/`get_component_relations` sitio por sitio. Mismos endpoints que `list_brick_vertices` (solo componentes con ≥1 relación). Mismo patrón de paginación. |
| `get_site_relations(client_id, site_id)` | Topología completa de UN sitio | `edges[]` es la única fuente viva aquí — los campos legados `outgoing_edges[]`/`incoming_edges[]`/`outgoing_count`/`incoming_count` están deprecados y SIEMPRE vacíos/0 en esta herramienta (a diferencia de `get_component_relations`, donde el equivalente sí está vivo). |
| `get_component_relations(client_id, component_id)` | Conexiones directas de UN dispositivo | Aquí `outgoing_edges[]`/`incoming_edges[]`/`outgoing_count`/`incoming_count` SÍ son el payload vivo (inverso de `get_site_relations`). Requiere primero `get_site_components` para obtener `component_id` (= `site_component_id`). |
| `get_component_relations_bulk(client_id, site_component_ids, direction?)` | Relaciones BRICS de varios componentes en una sola llamada | `direction` por defecto `"both"` en esta herramienta (distinto del default de upstream, que es `"out"` si se omite). `"out"` = solo relaciones donde el ID es origen — un receptor puro (ej. algo que solo recibe una arista `powers`) devuelve 0 filas aunque tenga relaciones reales; usar `"both"` o `"in"` para verlas. IDs deben ser enteros positivos separados por coma — tokens malformados → `VALIDATION_ERROR`; si todos son válidos pero ninguno es accesible para ese `client_id` → 422 de upstream. |

**Regla práctica:** si necesitas la topología de UN sitio o UN componente, usa
`get_site_relations`/`get_component_relations`. Si necesitas explorar o auditar
el grafo completo de un cliente (buscar patrones, tipos de arista, componentes
huérfanos), usa `list_brick_vertices`/`list_brick_edges` para no iterar sitio por sitio.

## 7. Directorio de acceso (no operativo)

| Herramienta | Qué es |
|---|---|
| `list_client_users(client_id)` | Directorio de personas con cuenta en la plataforma para ese cliente, con su rol. Es control de acceso, NO datos operativos — los usuarios nunca son referenciados por reglas, eventos ni el grafo BRICS, y no sirven como scope de ninguna otra herramienta. Sin paginación ni filtros — hacer match local por nombre/email. Es información personal: repórtala si el usuario la pide, no la muestres sin que la pidan. Para destinatarios de notificaciones de alertas, usar `get_site_rules`/`get_rule_custom_overrides`, no esto. |

## 8. Convenciones que rompen expectativas (checklist rápido)

- `component_id` significa DOS cosas distintas según el contexto: catálogo global de
  tipos (`list_type_components`) vs. dispositivo instalado (parámetro de
  `get_component_relations`/`get_component_rules`, que en realidad es un
  `site_component_id`). Verificar siempre cuál aplica antes de usarlo.
- Filtrar `get_events`/`get_alerts` solo por `site_component_id` activa una ventana
  implícita de 1 día — pasar `range_id` o `date_from`+`date_to` si se necesita más.
- `get_current_state.alerts[]` es top-N, no exhaustivo — comparar contra `open_alerts`
  antes de asumir que la lista está completa.
- `get_summary` nunca se suma entre filas (ventanas rolling que se solapan) y `period`
  no es un filtro de fecha.
- `get_measurements` con `raw=False` ignora `limit` y puede truncar silenciosamente
  (`window_truncated=True`) sin cubrir toda la ventana pedida.
- Campos deprecados que devuelven vacío/0 en una herramienta pero viven en otra
  (`outgoing_edges`/`incoming_edges`/`*_count`): vivos en `get_component_relations`,
  muertos en `get_site_relations`.
- `Rule.severity` no es la severidad real de alerta — la fuente de verdad es
  `get_rule_classifications`.
- Un resultado vacío de `find_assets` puede ser una falla parcial de upstream, no una
  ausencia real — revisar `partial` antes de concluir que algo no existe.
- 404 (`ENTITY_NOT_FOUND`) ≠ 200 con lista vacía: la primera es "no existe", la segunda
  es "existe pero sin datos que coincidan" — no tratarlas igual.

## 9. Encadenamiento típico

```
clients()
  → list_sites(client_id)
      → get_site_components(client_id, site_id)     # inventario
      → get_site_relations(client_id, site_id)       # topología del sitio
      → get_current_state(client_id, "site", site_id) # triage rápido
          → get_alerts / get_events (si se necesita el detalle completo)
              → get_component_rules(client_id, site_component_id)  # por qué se disparó
                  → get_rule_classifications(client_id, rule_id)   # severidad real
```

`find_assets` puede saltarse los primeros pasos cuando se conoce el nombre en texto
libre del recurso buscado, en vez de encadenar manualmente.
