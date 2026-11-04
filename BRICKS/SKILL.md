---
name: "bricks"
description: "Analiza relaciones Brick de Reveal (padre-hijo de datos/comunicación/control): estados, alertas que debieron suprimirse al caer un padre, auditoría de topología y de reglas de cascada."
---

# Bricks — relaciones padre-hijo y estados en Reveal

Úsala siempre que pregunten quién envía datos a quién, quién controla a quién, de quién depende la comunicación de un equipo, qué cae si cae un switch/gateway/DVR/appliance/medidor general, por qué hay alertas en cascada, si la supresión por padre (discon_parent_validation / parent_validation, reglas "Cascada" o "Sombra") está bien configurada, o pidan auditar la topología Brick, huérfanos, multi-padre o la jerarquía de un cliente, sitio o componente, aunque no digan "Brick".

Bricks responde cuatro preguntas sobre los componentes de un cliente en Reveal: quién envía datos a quién, quién recibe datos de quién, quién controla a quién y de quién depende el canal de comunicación de cada equipo. Con esa jerarquía asigna un estado a cada componente y decide qué alertas son legítimas y cuáles sobran.

Es **solo de consulta**: el MCP de Reveal no crea, cierra ni suprime alertas, y esta skill tampoco lo intenta. Señala lo que está mal y por qué.

## Regla de negocio (la razón de ser de la skill)

Cuando un padre pierde comunicación, **solo debe existir la alerta del padre**. Ningún descendiente (hijos, nietos, a cualquier profundidad) debe alertar a partir de ese momento, porque sin el padre no hay forma de saber qué pasa con ellos: el silencio del hijo es consecuencia, no falla.

- Ejemplo: Cámara → DVR. Si cae el DVR, solo alerta el DVR.
- Ejemplo: Switch → DVR → cámaras. Si cae el switch, solo alerta el switch.
- Ejemplo: Gateway → baños de María, extractores, sensores, medidores. Si cae el gateway, solo alerta el gateway.

Precisiones confirmadas por el usuario:

1. La supresión baja en cascada a todos los niveles.
2. Un componente tiene **un solo padre de comunicación**. Dos padres es un error de topología.
3. Solo la **pérdida de comunicación** del padre suprime. Un DVR con disco lleno o con la hora atrasada no suprime nada.
4. **Solo sobran las alertas de descendientes emitidas o creadas *después* de que el padre perdió comunicación.** Una alerta del hijo que ya existía con el padre en línea es comportamiento normal: se omite. Se reporta como «Válida (previa a la caída del padre)», no como error.
5. Si el padre no tiene falla de comunicación, el hijo puede generar cualquier alerta.

**Inicio de la caída del padre.** Es la marca de tiempo más temprana de su alerta de comunicación (`event_at`, `created_at` o `activated_at`), o el `created_at` de su notificación *pending* de regla sombra. Usa la misma zona horaria para todo; lo más seguro son los campos `*_utc`.

**Carrera de tiempos.** En caídas reales de Ladrillera, los hijos quedaron *unreachable* entre 5 y 77 s **antes** que el switch padre. Cuando el hijo evaluó `parent_validation`, el padre aún no figuraba caído y la alerta salió. Por la regla 4 no cuentan como «Debió suprimirse», pero tampoco son normales. El script las marca como **«Carrera de tiempos (revisar)»** si se emitieron hasta 120 s antes de la caída del padre (se cambia con `--ventana-carrera`). Repórtalas aparte: muestran que la supresión de Reveal depende del orden en que llegan los eventos y no de la topología.

## Leer bien las aristas: la frase manda

Cada arista del MCP viene como `from_vertex_id → label → to_vertex_id`. **Léela siempre como una frase**: «*from* `label` *to*». El padre sale de esa frase, no de la posición. La misma flecha significa jerarquías opuestas según el verbo. Es el error más caro de la skill, así que nunca asumas la dirección.

| Etiqueta (vistas en Reveal) | Frase | Padre | ¿Suprime? |
|---|---|---|---|
| `sendsDataTo` | A envía datos a B | **B** (destino) | Sí, es comunicación |
| `hasSubMeter` | A tiene como submedidor a B | **A** (origen) | Sí, el submedidor se comunica a través del general |

- `Cámara sendsDataTo DVR sendsDataTo SW-Almacen sendsDataTo SW-Admin sendsDataTo Appliance virtual`: la flecha va del hijo al padre y la raíz es el appliance.
- `Medidor GENERAL hasSubMeter Medidor MOLDEO`: la flecha va del **padre** al hijo. Si cae el GENERAL, solo alerta el GENERAL.

Las dos etiquetas se suman para la regla de un solo padre: un submedidor con `hasSubMeter` desde un general y además `sendsDataTo` hacia otro equipo tiene dos padres.

El script (Anexo A) trae además la lectura de otras etiquetas Brick habituales (`receivesDataFrom`, `controls`/`isControlledBy`, `feeds`/`isFedBy`, `powers`, `hasPart`/`isPartOf`…). Solo las de comunicación propagan supresión. Las de control, suministro y estructura se reportan como jerarquía, pero no suprimen.

**Etiqueta nueva o dudosa:** léela como frase, explica tu lectura y pregúntale al usuario antes de usarla para suprimir. El script la marca como «Etiqueta no catalogada». Cuando el usuario la confirme, pásala en `label_semantics` (ver Anexo B).

## Cómo funciona la supresión dentro de Reveal

Reveal ya implementa la supresión en las reglas. Entender esto permite auditarla:

- `rule.component_id` es el **tipo** de componente (catálogo), no un equipo. Ejemplos vistos: 15 DVR inhouse, 16 cámara análoga, 69 cámara IP, 103 NVR, 130 disco, 151 switch, 183 appliance virtual. Confírmalo con `get_site_components`, que devuelve `component_id` y `component_name` por equipo.
- `discon_parent_validation = 1`: la regla consulta al padre Brick antes de alertar.
- `parent_validation = "id,id,…"`: las reglas **del padre** que, si están abiertas o *pending* en el padre, impiden la alerta del hijo.
- **Reglas sombra** («Cascada - Sombra», `activate_on` enorme): nunca activan alerta. Solo dejan una notificación *pending* en el padre para que los hijos validen contra ella. No son un error, y esa notificación pending prueba que el padre está caído. Si no aparece ninguna pending de las sombras en semanas, vale la pena verificarlo con desarrollo, porque sin ellas la cascada no puede validar.
- Una regla de un tipo que tiene padre en el grafo y `discon_parent_validation = 0` (por ejemplo «Cámara Análoga 72 horas sin grabación») **puede alertar aunque el padre esté caído**. Reporta su severidad según si genera alerta (`get_rule_classifications`) o solo notificación.

**Qué es «pérdida de comunicación»:**
- **Sí lo son:** reglas `disconnection` con códigos de enlace (`unreachable`, `connectionFailed`, `Disconnected`, `loginFailed`) y reglas `data-sync` (timeout: el canal dejó de reportar).
- **No lo son:** fallas de disco (`disk_*`), sin grabación, hora desfasada, accesos, movimiento, anomalías ni mantenimiento.

El script clasifica con esa heurística y deja el motivo en la hoja «Reglas». Revísala, y si alguna regla quedó mal clasificada, corrígela con `comm_rule_ids` o `not_comm_rule_ids`.

## Estados y clasificación de alertas

| Estado del componente | Cuándo |
|---|---|
| `Falla raíz` | Tiene pérdida de comunicación propia y ningún ancestro en falla. |
| `Afectado por padre (suprimir)` | Algún ancestro está hoy en falla de comunicación. |
| `Falla propia` | Tiene alertas activas que no son de comunicación y su padre está bien. |
| `Sin datos` | Frescura `stale` sin alerta de comunicación: posible falla no detectada. |
| `Sin relaciones` | Está en el inventario del sitio pero no participa en ninguna arista. |
| `OK` | Nada de lo anterior. |

| Alerta activa | Cuándo |
|---|---|
| `Alerta raíz válida` | Es de comunicación y el componente no tiene ancestros caídos. |
| `Debió suprimirse` | Se emitió **después** de la caída de comunicación de un ancestro. |
| `Carrera de tiempos (revisar)` | Se emitió hasta 120 s **antes** de que el ancestro registrara su caída. |
| `Válida (previa a la caída del padre)` | Ya existía con el padre en línea. Es normal y se omite del conteo. |
| `Válida (padre sin falla)` | Ningún ancestro está caído. |

## Flujo de trabajo

### 1. Alcance y cliente
- El usuario puede pedir un cliente completo, uno o varios sitios, o un componente. Si no está claro, pregunta.
- Resuelve el `client_id` con `clients()` y no uses un ID recordado sin verificarlo. Resuelve el sitio con `list_sites` y el componente con `find_assets`.
- El `id` de un vértice Brick **es** el `site_component_id`. Se cruza directamente con `get_site_components`, `get_current_state` y `get_alerts`.

### 2. Recolectar (todo paginado hasta `has_more=false`)
| Dato | Llamada | Para qué |
|---|---|---|
| Aristas | `list_brick_edges(client_id, limit=1000)` | Grafo completo del cliente; las relaciones cruzan sitios |
| Vértices | `list_brick_vertices(client_id, limit=1000)` | Nombre, tipo y sitio de cada nodo |
| Inventario | `get_site_components(site_id, limit=1000)` por sitio | Tipo de catálogo (`component_id`) y huérfanos |
| Frescura | `get_current_state(scope_type="site")` por sitio | `live` / `stale` y el total `open_alerts` |
| Alertas abiertas | `get_alerts(site_id, statuses="active", limit=100)` por sitio, todas las páginas | Qué alertas existen, con `created_at`/`event_at`/`activated_at` (preferiblemente `*_utc`) |
| Reglas | `get_site_rules(site_id, limit=1000)` por sitio | Clasificación de comunicación y auditoría de cascada |
| Reglas que generan alerta | `get_rule_classifications(client_id)` | `alert_rule_ids`: separa reglas que alertan de las que solo notifican |
| Pending de sombras | `get_events(site_id, statuses="pending", rule_ids=<sombras y reglas de comunicación de padres>, range_id="30d")` | Padres caídos que aún no alertan |

Trampas de los datos:
- `get_alerts` aplica una ventana por defecto. Si el total no cuadra con `open_alerts` de `get_current_state`, amplía con `date_from` y `date_to` hasta que cuadre.
- `get_current_state(site)` trunca `data_freshness` a unos 100 componentes. Si viene truncado, pon `"freshness_truncated": true` y consulta `get_current_state(scope_type="component")` solo para los componentes con alertas y los padres en falla.
- El MCP limita a unas 60 llamadas por minuto. Agrupa con `get_component_relations_bulk` y evita las consultas uno a uno.
- Hay aristas que apuntan a componentes de sitios sin acceso para el cliente. Repórtalas como relaciones huérfanas.
- Para una pregunta sobre un solo componente basta `get_component_relations_bulk(direction="both")`, recorriendo hacia arriba y hacia abajo.
- Si el grafo es muy grande, filtra las aristas a los sitios del alcance más sus padres hacia arriba, aunque estén en otro sitio.

### 3. Analizar con el script
El script y el formato de entrada vienen en los anexos al final de este archivo. Si `scripts/bricks_analyzer.py` no existe junto a la skill, copia el **Anexo A** tal cual a `bricks_analyzer.py` en el directorio de trabajo (necesita `openpyxl`). Escribe los datos en un JSON compacto según el **Anexo B**. Las aristas van agrupadas por destino o por origen y el resto en listas, para no gastar tokens. Luego:

```bash
python bricks_analyzer.py entrada.json -o Bricks_<cliente>_<alcance>_<AAAAMMDD>.xlsx   # análisis completo
python bricks_analyzer.py entrada.json --simular <site_component_id>                 # ¿qué pasa si cae X?
python bricks_analyzer.py entrada.json --componente <site_component_id>              # relaciones y estado de X
```

El script calcula:
- jerarquía, ciclos y multi-padre;
- huérfanos, con **padre sugerido por nombre** (por ejemplo «DISCO 3 NVR GECOLSA TURBACO 3» → «NVR GECOLSA TURBACO 3»);
- nombres que no coinciden con su padre;
- alertas de comunicación abiertas en equipos que reportan `live`;
- estados, clasificación temporal de alertas y auditoría de cascada.

Úsalo en vez de razonar el grafo a mano, porque con cientos de aristas es fácil equivocarse. Lee el resumen JSON que imprime y revisa las hojas «Reglas» y «Relaciones» antes de concluir. Las sugerencias de padre son heurísticas y hay que validarlas antes de crear relaciones.

### 4. Entregar
- **Análisis de cliente o sitio** (estados, supresión, auditoría): el Excel con las hojas Resumen, Estados, Alertas, Jerarquía, Otras jerarquías, Hallazgos topología, Config cascada, Reglas y Relaciones. Envíalo y escribe en el chat de 3 a 6 líneas: fallas raíz, alertas que debieron suprimirse, carreras de tiempos, hallazgos altos y reglas que rompen la cascada. Si el usuario quiere verlo visual, ofrece un artefacto (dashboard) con los mismos datos.
- **Pregunta puntual** («¿de quién depende X?», «¿qué cae si cae Y?»): responde en el chat. Incluye la cadena `X → padre → … → raíz`, los hijos directos, los descendientes por tipo y cuántas alertas deberían existir (una, la del padre). Agrega qué reglas de los descendientes no suprimirían, separando las que generan alerta de las que solo notifican. No hagas Excel salvo que lo pidan.
- **«¿Por qué no se suprimió?» o simulación con historia:** revisa caídas reales anteriores del padre con `get_events(site_component_id=<padre>, statuses="active,closed", range_id="30d")` y compáralas con las alertas de los hijos en la misma ventana. Así se demuestra la carrera de tiempos con evidencia y no solo con la configuración.
- Cita nombres e IDs como vienen de Reveal (en español, sin traducir) y el `event_id` de cada alerta que menciones.
- Sé breve y directo: sin introducciones, solo lo consultado.

## Errores que hay que evitar
- Invertir padre e hijo por no leer la frase: `sendsDataTo` apunta al padre y `hasSubMeter` sale del padre.
- Contar como «Debió suprimirse» una alerta del hijo que ya existía antes de la caída del padre.
- Concluir «sin alertas» con una sola página de `get_alerts`, sin cuadrar contra `open_alerts`, o sin `range_id` en `get_events`.
- Dar por bueno al padre solo porque no tiene alerta activa, sin revisar las *pending* de las sombras ni la frescura.
- Contar como falla de comunicación del padre una alerta que no es de comunicación (disco, grabación, hora).
- Tratar la regla sombra como mal configurada.
- Reportar N fallas independientes cuando muchos equipos de un sitio caen a la vez: primero busca el padre común.
- Llamar a `logout()`. Nunca.

---

## Anexo A — `bricks_analyzer.py`

```python
#!/usr/bin/env python3
"""
Bricks — analizador de relaciones padre-hijo entre componentes de Reveal.

Lee un JSON con los datos ya consultados al MCP de Reveal (vértices, aristas,
inventario, frescura, alertas, notificaciones pending y reglas) y produce:
  * un Excel con estados, jerarquía, clasificación de alertas, hallazgos de
    topología y auditoría de la configuración de cascada;
  * un resumen JSON por stdout (para redactar la respuesta en el chat).

Uso:
  python bricks_analyzer.py entrada.json -o salida.xlsx
  python bricks_analyzer.py entrada.json --simular 37584        # ¿qué pasa si cae X?
  python bricks_analyzer.py entrada.json --componente 32087     # cadena y relaciones de X

Formato de entrada: ver Anexo B del SKILL.md de bricks.
"""
import argparse
import json
import re
import sys
import unicodedata
from datetime import datetime, timezone
from collections import defaultdict, Counter

# ---------------------------------------------------------------------------
# Semántica de etiquetas. Se lee la arista como frase: "<from> <label> <to>".
#   parent_end: qué extremo es el padre ("to" o "from")
#   kind: tipo de dependencia
#   propagates: si la pérdida de comunicación del padre suprime alertas del hijo
# ---------------------------------------------------------------------------
LABELS = {
    "sendsDataTo":       {"parent_end": "to",   "kind": "comunicacion", "propagates": True,
                          "frase": "{a} envía datos a {b}"},
    "receivesDataFrom":  {"parent_end": "from", "kind": "comunicacion", "propagates": True,
                          "frase": "{a} recibe datos de {b}"},
    "isConnectedThrough":{"parent_end": "to",   "kind": "comunicacion", "propagates": True,
                          "frase": "{a} se comunica a través de {b}"},
    "providesConnectivityTo": {"parent_end": "from", "kind": "comunicacion", "propagates": True,
                          "frase": "{a} da comunicación a {b}"},
    "controls":          {"parent_end": "from", "kind": "control", "propagates": False,
                          "frase": "{a} controla a {b}"},
    "isControlledBy":    {"parent_end": "to",   "kind": "control", "propagates": False,
                          "frase": "{a} es controlado por {b}"},
    # Confirmado por el usuario: el submedidor se comunica a través del medidor general.
    "hasSubMeter":       {"parent_end": "from", "kind": "comunicacion", "propagates": True,
                          "frase": "{a} tiene como submedidor a {b} ({b} se comunica a través de {a})"},
    "isSubMeterOf":      {"parent_end": "to",   "kind": "comunicacion", "propagates": True,
                          "frase": "{a} es submedidor de {b} ({a} se comunica a través de {b})"},
    "feeds":             {"parent_end": "from", "kind": "suministro", "propagates": False,
                          "frase": "{a} alimenta a {b}"},
    "isFedBy":           {"parent_end": "to",   "kind": "suministro", "propagates": False,
                          "frase": "{a} es alimentado por {b}"},
    "powers":            {"parent_end": "from", "kind": "suministro", "propagates": False,
                          "frase": "{a} energiza a {b}"},
    "isPoweredBy":       {"parent_end": "to",   "kind": "suministro", "propagates": False,
                          "frase": "{a} es energizado por {b}"},
    "suppliesFuelTo":    {"parent_end": "from", "kind": "suministro", "propagates": False,
                          "frase": "{a} suministra combustible a {b}"},
    "hasPart":           {"parent_end": "from", "kind": "estructura", "propagates": False,
                          "frase": "{a} contiene a {b}"},
    "isPartOf":          {"parent_end": "to",   "kind": "estructura", "propagates": False,
                          "frase": "{a} es parte de {b}"},
}

ROOT_HINTS = ("appliance", "gateway")  # tipos que pueden ser raíz de comunicación sin padre
COMM_CODES = ("unreachable", "connectionfailed", "disconnected", "loginfailed")
NON_COMM_CODES = ("disk_", "moton", "anmdet", "loclog", "illacc", "curdaymtto", "mttosit")


# ---------------------------------------------------------------------------
# Carga flexible (listas compactas o dicts)
# ---------------------------------------------------------------------------
def rows(data, key, fields):
    out = []
    for r in data.get(key, []) or []:
        if isinstance(r, dict):
            out.append({f: r.get(f) for f in fields} | {k: v for k, v in r.items() if k not in fields})
        else:
            d = {f: (r[i] if i < len(r) else None) for i, f in enumerate(fields)}
            out.append(d)
    return out


def expand_grouped(data):
    """edges_by_target: {"sendsDataTo": {"<to>": [from, ...]}} -> aristas from->to.
    edges_by_source: {"hasSubMeter": {"<from>": [to, ...]}} -> aristas from->to."""
    out = []
    for lab, m in (data.get("edges_by_target") or {}).items():
        for to, frs in m.items():
            out += [{"id": None, "from": f, "to": int(to), "label": lab} for f in frs]
    for lab, m in (data.get("edges_by_source") or {}).items():
        for fr, tos in m.items():
            out += [{"id": None, "from": int(fr), "to": t, "label": lab} for t in tos]
    return out


def parse_ts(x):
    """Acepta 'YYYY-MM-DD HH:MM:SS' o ISO con zona. Devuelve datetime naive en UTC si trae zona."""
    if not x:
        return None
    s = str(x).strip().replace("Z", "+00:00")
    try:
        d = datetime.fromisoformat(s.replace(" ", "T") if "T" not in s else s)
    except ValueError:
        return None
    if d.tzinfo:
        d = d.astimezone(timezone.utc).replace(tzinfo=None)
    return d


def to_int(x):
    try:
        return int(x)
    except (TypeError, ValueError):
        return None


def load(path):
    with open(path, encoding="utf-8") as fh:
        data = json.load(fh)
    D = {
        "client": data.get("client", {}),
        "scope": data.get("scope", {}),
        "generated_at": data.get("generated_at"),
        "vertices": rows(data, "vertices", ["id", "label", "source_type", "system", "site_id"]),
        "edges": rows(data, "edges", ["id", "from", "to", "label"]) + expand_grouped(data),
        "components": rows(data, "components", ["id", "site_id", "type_id", "type_name", "description"]),
        "freshness": rows(data, "freshness", ["id", "status", "last_seen"]),
        "alerts": rows(data, "alerts", ["event_id", "site_component_id", "rule_id", "falla",
                                         "severity", "activated_at", "status", "created_at", "event_at"]),
        "pending": rows(data, "pending", ["event_id", "site_component_id", "rule_id", "falla", "created_at"]),
        "rules": rows(data, "rules", ["id", "name", "type", "type_component_id",
                                       "discon_parent_validation", "parent_validation",
                                       "disconnection_codes", "status", "activate_on"]),
        "comm_rule_ids": set(int(x) for x in data.get("comm_rule_ids", []) or []),
        "not_comm_rule_ids": set(int(x) for x in data.get("not_comm_rule_ids", []) or []),
        "alert_rule_ids": set(int(x) for x in data.get("alert_rule_ids", []) or []),
        "freshness_truncated": bool(data.get("freshness_truncated", False)),
        "label_semantics": data.get("label_semantics", {}) or {},
    }
    # normalizar ids
    for v in D["vertices"]:
        v["id"] = to_int(v["id"]); v["site_id"] = to_int(v["site_id"])
    for e in D["edges"]:
        e["from"] = to_int(e["from"]); e["to"] = to_int(e["to"])
    for c in D["components"]:
        c["id"] = to_int(c["id"]); c["site_id"] = to_int(c["site_id"]); c["type_id"] = to_int(c["type_id"])
    for f in D["freshness"]:
        f["id"] = to_int(f["id"])
    for a in D["alerts"] + D["pending"]:
        a["site_component_id"] = to_int(a["site_component_id"]); a["rule_id"] = to_int(a["rule_id"])
    for r in D["rules"]:
        r["id"] = to_int(r["id"]); r["type_component_id"] = to_int(r["type_component_id"])
        r["discon_parent_validation"] = to_int(r["discon_parent_validation"]) or 0
        r["activate_on"] = to_int(r.get("activate_on")) or 0
        r["sombra"] = "sombra" in (r.get("name") or "").lower() or r["activate_on"] >= 525600
        pv = r.get("parent_validation") or ""
        r["parent_validation"] = [int(x) for x in re.findall(r"\d+", str(pv))]
    return D


# ---------------------------------------------------------------------------
# Clasificación de reglas de pérdida de comunicación
# ---------------------------------------------------------------------------
STOP = {"dvr", "nvr", "de", "la", "el", "los", "las", "del", "y", "canal", "camera", "camara", "cam", "ip", "disco"}


def words(s):
    s = unicodedata.normalize("NFKD", s or "").encode("ascii", "ignore").decode().lower()
    return {w for w in re.findall(r"[a-z0-9]+", s) if w not in STOP}


def classify_rule(r, D):
    rid = r["id"]
    if rid in D["comm_rule_ids"]:
        return True, "Indicada explícitamente como comunicación"
    if rid in D["not_comm_rule_ids"]:
        return False, "Indicada explícitamente como NO comunicación"
    name = (r.get("name") or "").lower()
    codes = (r.get("disconnection_codes") or "").lower()
    rtype = (r.get("type") or "").lower()
    if any(c in codes for c in NON_COMM_CODES):
        return False, f"Códigos no de comunicación ({codes})"
    if "mantenimiento" in name or "anomal" in name or "acceso" in name or "movimiento" in name:
        return False, "Nombre indica evento operativo, no pérdida de comunicación"
    if rtype == "disconnection" and any(c in codes.replace(" ", "") for c in COMM_CODES):
        return True, f"disconnection con códigos de enlace ({codes})"
    if rtype == "data-sync":
        return True, "data-sync (timeout de datos = canal sin reportar)"
    if rtype == "disconnection" and re.search(r"comunicaci|desconectad|conexi|login", name):
        return True, "disconnection con nombre de conexión/comunicación"
    return False, f"Tipo {rtype or '—'} no es pérdida de comunicación"


# ---------------------------------------------------------------------------
# Análisis
# ---------------------------------------------------------------------------
def analyze(D):
    V = {v["id"]: v for v in D["vertices"]}
    C = {c["id"]: c for c in D["components"]}
    label_of = lambda i: (V.get(i) or {}).get("label") or (C.get(i) or {}).get("description") or f"#{i}"
    type_of = lambda i: (V.get(i) or {}).get("source_type") or (C.get(i) or {}).get("type_name") or ""
    site_of = lambda i: (V.get(i) or {}).get("site_id") or (C.get(i) or {}).get("site_id")

    sem = dict(LABELS)
    for k, v in D["label_semantics"].items():
        sem[k] = {"parent_end": v.get("parent_end", "to"), "kind": v.get("kind", "otro"),
                  "propagates": bool(v.get("propagates", False)),
                  "frase": v.get("frase", "{a} " + k + " {b}")}

    findings = []

    def add(sev, tipo, comp, detalle):
        findings.append({"severidad": sev, "tipo": tipo, "componente_id": comp,
                         "componente": label_of(comp) if comp else "", "detalle": detalle})

    # --- relaciones con lectura en frase ----------------------------------
    rel_rows = []
    parents = defaultdict(lambda: defaultdict(list))  # kind -> child -> [parents]
    children = defaultdict(lambda: defaultdict(list))  # kind -> parent -> [children]
    seen = set()
    labels_found = Counter()
    for e in D["edges"]:
        a, b, lab = e["from"], e["to"], e["label"]
        labels_found[lab] += 1
        s = sem.get(lab)
        if a == b:
            add("Alta", "Auto-relación", a, f"Arista {e['id']} '{lab}' apunta al mismo componente")
            continue
        key = (a, b, lab)
        if key in seen:
            add("Media", "Arista duplicada", a, f"{label_of(a)} {lab} {label_of(b)} repetida (arista {e['id']})")
        seen.add(key)
        if not s:
            rel_rows.append({"arista": e["id"], "from": a, "from_label": label_of(a), "label": lab,
                             "to": b, "to_label": label_of(b), "frase": f"{label_of(a)} {lab} {label_of(b)}",
                             "tipo": "NO CATALOGADA", "padre": "", "hijo": "", "propaga": "No"})
            continue
        p, c = (b, a) if s["parent_end"] == "to" else (a, b)
        parents[s["kind"]][c].append(p)
        children[s["kind"]][p].append(c)
        rel_rows.append({"arista": e["id"], "from": a, "from_label": label_of(a), "label": lab,
                         "to": b, "to_label": label_of(b),
                         "frase": s["frase"].format(a=label_of(a), b=label_of(b)),
                         "tipo": s["kind"], "padre": f"{p} · {label_of(p)}", "hijo": f"{c} · {label_of(c)}",
                         "propaga": "Sí" if s["propagates"] else "No"})
        sa, sb = site_of(a), site_of(b)
        if sa and sb and sa != sb:
            add("Baja", "Relación entre sitios", a, f"{label_of(a)} (sitio {sa}) {lab} {label_of(b)} (sitio {sb})")

    for lab in labels_found:
        if lab not in sem:
            add("Alta", "Etiqueta no catalogada", None,
                f"'{lab}' ({labels_found[lab]} aristas): confirmar dirección padre/hijo y si es de comunicación antes de usarla")

    # --- regla: un solo padre por tipo de dependencia ------------------------
    for kind, m in parents.items():
        for c, ps in m.items():
            ups = sorted(set(ps))
            if len(ups) > 1:
                add("Alta", f"Multi-padre ({kind})", c,
                    "Tiene más de un padre: " + "; ".join(f"{p} · {label_of(p)}" for p in ups))

    comm_kinds = {s["kind"] for s in sem.values() if s["propagates"]}
    comm_parent = {}
    comm_children = defaultdict(list)
    for kind in comm_kinds:
        for c, ps in parents[kind].items():
            comm_parent.setdefault(c, sorted(set(ps))[0])  # si hay multi-padre, se toma el menor y queda el hallazgo
        for p, cs in children[kind].items():
            comm_children[p].extend(cs)

    # --- ciclos -------------------------------------------------------------
    def chain(i):
        out, cur, guard = [], i, set()
        while cur in comm_parent:
            nxt = comm_parent[cur]
            if nxt in guard or nxt == i:
                return out, True
            guard.add(nxt); out.append(nxt); cur = nxt
        return out, False

    in_graph = set(V) | {e["from"] for e in D["edges"]} | {e["to"] for e in D["edges"]}
    universe = in_graph | set(C)
    cyc_reported = set()
    for i in universe:
        ch, cyc = chain(i)
        if cyc and i not in cyc_reported:
            cyc_reported |= set(ch) | {i}
            add("Alta", "Ciclo de comunicación", i, "Cadena circular: " + " → ".join(label_of(x) for x in [i] + ch))

    # --- sugerencia de padre por nombre --------------------------------------
    parent_pool = defaultdict(list)  # sitio -> posibles padres (ya son padres de alguien)
    for p in comm_children:
        parent_pool[site_of(p)].append(p)

    def suggest_parent(i):
        # "DISCO 3 NVR X" -> el número es del disco, no del grabador
        w = words(re.sub(r"(?i)\bdisco\s*(duro)?\s*\d+", " ", label_of(i) or ""))
        best, score, tie = None, (0.0, 0), False
        for p in parent_pool.get(site_of(i), []):
            if p == i:
                continue
            wp = words(label_of(p))
            if not wp:
                continue
            inter = len(w & wp)
            sc = (inter / len(wp), inter)  # cobertura del nombre del padre, luego palabras en común
            if sc > score:
                best, score, tie = p, sc, False
            elif sc == score and inter > 0:
                tie = True
        if best and score[0] >= 0.6 and not tie:
            return best
        return None

    suggestions = {}

    # --- raíces y huérfanos -------------------------------------------------
    for i in sorted(universe):
        has_p = i in comm_parent
        has_c = bool(comm_children.get(i))
        t = (type_of(i) or "").lower()
        lab = (label_of(i) or "").lower()
        is_root_type = any(h in t or h in lab for h in ROOT_HINTS) or (i in V and not V[i].get("source_type"))
        if i not in in_graph or (not has_p and not is_root_type):
            sp = suggest_parent(i)
            if sp:
                suggestions[i] = sp
            sug = f" · Padre sugerido por nombre: {sp} · {label_of(sp)} (validar antes de crear la relación)" if sp else ""
        if i not in in_graph:
            add("Media", "Sin relaciones", i,
                f"Está en el inventario del sitio {site_of(i)} pero no participa en ninguna relación Brick{sug}")
        elif not has_p and not is_root_type:
            other = [k for k in parents if i in parents[k]]
            extra = f" (solo tiene padre de {', '.join(other)})" if other else ""
            add("Media", "Sin padre de comunicación", i,
                f"No tiene canal de comunicación definido{extra}; si cae su enlace real, nadie suprime sus alertas{sug}")

    # --- consistencia de nombres (heurística CCTV) ---------------------------
    for c, p in comm_parent.items():
        tc = (type_of(c) or "").lower()
        if not any(k in tc for k in ("camara", "cámara", "video", "ip camera")):
            continue
        m = re.search(r"[DN]VR\s*-\s*(.+?)(?:\s+Canal|_|\s+-\s+|$)", label_of(c) or "")
        tp = (type_of(p) or "").lower()
        if m and ("dvr" in tp or "nvr" in tp):
            token = words(m.group(1))
            wp = words(label_of(p))
            if token and wp and len(token & wp) / len(wp) < 0.5:
                add("Baja", "Nombre no coincide con padre", c,
                    f"El nombre sugiere '{m.group(1).strip()}' pero el padre es '{label_of(p)}'")

    # --- reglas de comunicación ---------------------------------------------
    rules = {r["id"]: r for r in D["rules"] if r["id"] is not None}
    rule_class = {}
    for rid, r in rules.items():
        rule_class[rid] = classify_rule(r, D)
    is_comm = lambda rid: rule_class.get(rid, (rid in D["comm_rule_ids"], ""))[0]

    # --- fallas de comunicación por componente ------------------------------
    fresh = {f["id"]: f for f in D["freshness"]}
    alerts_by = defaultdict(list)
    for a in D["alerts"]:
        alerts_by[a["site_component_id"]].append(a)
    comm_down = {}

    def mark_down(cid, ts_list, evid, via):
        ts = [t for t in (parse_ts(x) for x in ts_list) if t]
        t = min(ts) if ts else None
        cur = comm_down.get(cid)
        if cur is None or (t and (cur["desde"] is None or t < cur["desde"])):
            comm_down[cid] = {"desde": t, "evidencia": evid, "via": via}

    for a in D["alerts"]:
        if is_comm(a["rule_id"]):
            mark_down(a["site_component_id"], [a.get("event_at"), a.get("created_at"), a.get("activated_at")],
                      f"Alerta {a['event_id']} · {a.get('falla') or a['rule_id']}", "alerta")
    for pn in D["pending"]:
        if is_comm(pn["rule_id"]) or pn["rule_id"] in D["comm_rule_ids"]:
            mark_down(pn["site_component_id"], [pn.get("created_at")],
                      f"Pending {pn['event_id']} · {pn.get('falla') or pn['rule_id']}", "pending")
    for cid, d in comm_down.items():
        if d["via"] == "alerta" and (fresh.get(cid, {}).get("status") or "").lower() == "live":
            add("Media", "Alerta de comunicación abierta con datos live", cid,
                f"{d['evidencia']} sigue activa pero el componente reporta datos"
                + (f" (último dato {fresh[cid].get('last_seen')})" if fresh[cid].get('last_seen') else "") + ": "
                "verificar si la caída sigue vigente o si la reconexión no cerró la alerta. Mientras siga abierta, "
                "Reveal lo trata como caído y sus hijos quedan en supresión")

    def failing_root(i):
        """Ancestro más alto con pérdida de comunicación (la causa raíz)."""
        anc, _ = chain(i)
        root = None
        for x in anc:
            if x in comm_down:
                root = x
        return root

    def descendants(i):
        out, stack, seen_ = [], list(comm_children.get(i, [])), {i}
        while stack:
            x = stack.pop()
            if x in seen_:
                continue
            seen_.add(x); out.append(x); stack.extend(comm_children.get(x, []))
        return out

    # --- estados -----------------------------------------------------------
    state_rows = []
    for i in sorted(universe):
        anc, _ = chain(i)
        root = failing_root(i)
        al = alerts_by.get(i, [])
        f = fresh.get(i, {})
        if root is not None:
            estado = "Afectado por padre (suprimir)"
        elif i in comm_down:
            estado = "Falla raíz"
        elif al:
            estado = "Falla propia"
        elif (f.get("status") or "").lower() == "stale":
            estado = "Sin datos"
        elif i not in in_graph:
            estado = "Sin relaciones"
        else:
            estado = "OK"
        p = comm_parent.get(i)
        state_rows.append({
            "id": i, "componente": label_of(i), "tipo": type_of(i), "sitio": site_of(i),
            "padre_id": p, "padre": label_of(p) if p else "",
            "cadena": " → ".join([label_of(i)] + [label_of(x) for x in anc]),
            "nivel": len(anc), "hijos_directos": len(set(comm_children.get(i, []))),
            "descendientes": len(descendants(i)),
            "falla_comunicacion": "Sí" if i in comm_down else "No",
            "evidencia_falla": comm_down.get(i, {}).get("evidencia", ""),
            "frescura": f.get("status") or "", "ultimo_dato": f.get("last_seen") or "",
            "alertas_activas": len(al), "estado": estado,
            "raiz_falla": f"{root} · {label_of(root)}" if root else "",
        })

    # --- clasificación de alertas -------------------------------------------
    alert_rows = []
    ventana = D.get("ventana_carrera_s", 120)
    for a in D["alerts"]:
        cid = a["site_component_id"]
        com = is_comm(a["rule_id"])
        anc, _ = chain(cid)
        down_anc = [x for x in anc if x in comm_down]
        t_child = parse_ts(a.get("activated_at")) or parse_ts(a.get("created_at"))
        after = [x for x in down_anc if comm_down[x]["desde"] and t_child and t_child >= comm_down[x]["desde"]]
        race = [x for x in down_anc if comm_down[x]["desde"] and t_child and
                0 < (comm_down[x]["desde"] - t_child).total_seconds() <= ventana]
        unknown = [x for x in down_anc if not (comm_down[x]["desde"] and t_child)]
        momento, raiz = "", ""
        if after:
            root = after[-1]  # el más alto ya caído cuando se emitió la alerta
            cls = "Debió suprimirse"
            raiz = f"{root} · {label_of(root)}"
            momento = f"Emitida {fmt_delta(t_child - comm_down[root]['desde'])} después de la caída del padre"
        elif race:
            root = race[-1]
            cls = "Carrera de tiempos (revisar)"
            raiz = f"{root} · {label_of(root)}"
            momento = (f"Emitida {fmt_delta(comm_down[root]['desde'] - t_child)} ANTES de que el padre registrara "
                       "su caída: probable misma caída; la validación de padre no alcanzó a actuar")
        elif unknown:
            root = unknown[-1]
            cls = "Sin fecha para comparar (revisar)"
            raiz = f"{root} · {label_of(root)}"
            momento = "Falta la fecha de la alerta o de la caída del padre"
        elif down_anc:
            root = down_anc[-1]
            cls = "Válida (previa a la caída del padre)"
            raiz = f"{root} · {label_of(root)}"
            momento = "Se emitió con el padre en línea: comportamiento normal, se omite"
        elif com:
            cls = "Alerta raíz válida"
        else:
            cls = "Válida (padre sin falla)"
        r = rules.get(a["rule_id"], {})
        alert_rows.append({
            "event_id": a["event_id"], "componente_id": cid, "componente": label_of(cid),
            "rule_id": a["rule_id"], "falla": a.get("falla") or r.get("name") or "",
            "severidad": a.get("severity") or "", "activada": a.get("activated_at") or "",
            "es_comunicacion": "Sí" if com else "No", "clasificacion": cls, "raiz": raiz, "momento": momento,
            "regla_valida_padre": ("Sí" if r.get("discon_parent_validation") else "No") if r else "¿?",
        })

    # --- auditoría de configuración de cascada ------------------------------
    cfg_rows = []
    type_by_comp = {i: (C.get(i) or {}).get("type_id") for i in universe}
    parent_types = defaultdict(set)  # tipo hijo -> tipos padre
    for c, p in comm_parent.items():
        tc, tp = type_by_comp.get(c), type_by_comp.get(p)
        if tc and tp:
            parent_types[tc].add(tp)
    type_name = {}
    for c in D["components"]:
        if c["type_id"]:
            type_name[c["type_id"]] = c.get("type_name") or str(c["type_id"])
    if not D["components"]:
        add("Info", "Auditoría de cascada omitida", None,
            "No se entregó inventario (get_site_components) con component_id de tipo; no se puede cruzar reglas con tipos")
    for rid, r in sorted(rules.items()):
        if (r.get("status") or "active") != "active":
            continue
        t = r["type_component_id"]
        if t not in parent_types:
            continue
        ptypes = parent_types[t]
        com, why = rule_class[rid]
        refs = r["parent_validation"]
        missing = [x for x in refs if x not in rules]
        covers = [x for x in refs if x in rules and rules[x]["type_component_id"] in ptypes and is_comm(x)]
        if r["sombra"]:
            hall = "Regla sombra: nunca activa alerta; solo deja la notificación pending contra la que validan los hijos"
            sev = "Info"
        elif not r["discon_parent_validation"]:
            hall = "NO valida al padre: si el padre pierde comunicación, esta regla igual genera alertas en el hijo"
            sev = "Alta"
        elif not refs:
            hall = "discon_parent_validation=1 pero parent_validation vacío: no hay regla del padre contra la cual validar"
            sev = "Alta"
        elif not covers:
            hall = ("Ninguna regla referenciada es de comunicación sobre el tipo del padre"
                    + (f" (no encontradas en el sitio: {missing})" if missing else ""))
            sev = "Alta" if not missing else "Media"
        else:
            hall = "OK" + (f" (referencias no encontradas en el sitio: {missing}; verificar a nivel cliente/componente)" if missing else "")
            sev = "Info" if missing else "OK"
        gen = ("Sí" if rid in D["alert_rule_ids"] else "No") if D["alert_rule_ids"] else "¿?"
        if sev == "Alta" and gen == "No":
            sev = "Media"
            hall += " · Solo genera notificación (no está clasificada como alerta)"
        cfg_rows.append({
            "rule_id": rid, "regla": r.get("name"), "tipo_hijo": f"{t} · {type_name.get(t, '')}",
            "genera_alerta": gen,
            "tipos_padre": ", ".join(f"{x} · {type_name.get(x, '')}" for x in sorted(ptypes)),
            "es_comunicacion": "Sí" if com else "No",
            "discon_parent_validation": r["discon_parent_validation"],
            "parent_validation": ",".join(map(str, refs)), "cubre_padre_con": ",".join(map(str, covers)),
            "severidad": sev, "hallazgo": hall,
        })

    rule_rows = [{"rule_id": rid, "regla": r.get("name"), "tipo": r.get("type"),
                  "tipo_componente": f"{r['type_component_id']} · {type_name.get(r['type_component_id'], '')}",
                  "codigos": r.get("disconnection_codes") or "",
                  "es_comunicacion": "Sí" if rule_class[rid][0] else "No", "motivo": rule_class[rid][1]}
                 for rid, r in sorted(rules.items())]

    # --- jerarquía (árbol) ---------------------------------------------------
    tree_rows = []
    roots = [i for i in sorted(in_graph) if i not in comm_parent and comm_children.get(i)]

    def walk(i, lvl, seen_):
        if i in seen_:
            return
        seen_.add(i)
        st = next((s["estado"] for s in state_rows if s["id"] == i), "")
        tree_rows.append({"nivel": lvl, "arbol": ("    " * lvl) + ("└─ " if lvl else "") + label_of(i),
                          "id": i, "tipo": type_of(i), "estado": st})
        for c in sorted(set(comm_children.get(i, [])), key=lambda x: label_of(x)):
            walk(c, lvl + 1, seen_)
    seen_tree = set()
    for r in roots:
        walk(r, 0, seen_tree)
    # jerarquías no de comunicación (control, suministro, estructura)
    other_tree = []
    for kind, m in children.items():
        if kind in comm_kinds:
            continue
        for p in sorted(m):
            if p in parents[kind]:
                continue
            def w(i, lvl, sn):
                if i in sn: return
                sn.add(i)
                other_tree.append({"jerarquia": kind, "nivel": lvl,
                                   "arbol": ("    " * lvl) + ("└─ " if lvl else "") + label_of(i), "id": i})
                for c in sorted(set(m.get(i, []))):
                    w(c, lvl + 1, sn)
            w(p, 0, set())

    ctx = dict(V=V, C=C, label_of=label_of, comm_parent=comm_parent, comm_children=comm_children,
               chain=chain, descendants=descendants, alerts_by=alerts_by, rules=rules,
               is_comm=is_comm, comm_down=comm_down, parents=parents, children=children, sem=sem)
    return dict(findings=findings, rel_rows=rel_rows, state_rows=state_rows, alert_rows=alert_rows,
                cfg_rows=cfg_rows, rule_rows=rule_rows, tree_rows=tree_rows, other_tree=other_tree,
                labels_found=labels_found, ctx=ctx, sem=sem)


# ---------------------------------------------------------------------------
# Consultas puntuales
# ---------------------------------------------------------------------------
def simulate(R, cid):
    x = R["ctx"]; lab = x["label_of"]
    desc = x["descendants"](cid)
    anc, _ = x["chain"](cid)
    tipos = Counter((x["V"].get(d) or {}).get("source_type") or (x["C"].get(d) or {}).get("type_name") or "sin tipo"
                    for d in desc)
    dtypes = {(x["C"].get(d) or {}).get("type_id") for d in desc} - {None}
    no_supr = [f"{c['rule_id']} · {c['regla']} ({c['tipo_hijo']})" for c in R["cfg_rows"]
               if c["severidad"] in ("Alta", "Media") and to_int(c["tipo_hijo"].split(" ")[0]) in dtypes]
    activas = [f"{a['event_id']} · {lab(a['site_component_id'])} · {a.get('falla')}"
               for d in desc for a in x["alerts_by"].get(d, [])]
    return {
        "componente": f"{cid} · {lab(cid)}",
        "cadena_hacia_arriba": [f"{a} · {lab(a)}" for a in anc],
        "alertas_esperadas": 1,
        "mensaje": f"Si {lab(cid)} pierde comunicación, solo debe existir la alerta de {lab(cid)}; "
                   f"{len(desc)} descendientes quedan 'Afectado por padre (suprimir)'.",
        "descendientes_afectados": len(desc),
        "por_tipo": dict(tipos),
        "hijos_directos": [f"{c} · {lab(c)}" for c in sorted(set(x["comm_children"].get(cid, [])))],
        "reglas_que_no_suprimirian": no_supr,
        "alertas_activas_hoy_en_descendientes": activas,
        "descendientes": [f"{d} · {lab(d)}" for d in desc],
    }


def component_view(R, cid):
    x = R["ctx"]; lab = x["label_of"]
    rels = [r for r in R["rel_rows"] if r["from"] == cid or r["to"] == cid]
    st = next((s for s in R["state_rows"] if s["id"] == cid), {})
    return {"componente": f"{cid} · {lab(cid)}", "estado": st,
            "relaciones": [{"frase": r["frase"], "tipo": r["tipo"], "padre": r["padre"], "hijo": r["hijo"]} for r in rels],
            "hallazgos": [f for f in R["findings"] if f["componente_id"] == cid]}


# ---------------------------------------------------------------------------
# Excel
# ---------------------------------------------------------------------------
def fmt_delta(td):
    s = int(abs(td.total_seconds()))
    if s < 120:
        return f"{s} s"
    if s < 7200:
        return f"{s // 60} min"
    if s < 172800:
        return f"{s / 3600:.1f} h"
    return f"{s / 86400:.1f} d"


def write_excel(R, D, path):
    from openpyxl import Workbook
    from openpyxl.styles import Font, PatternFill, Alignment
    from openpyxl.utils import get_column_letter

    wb = Workbook()
    HDR = PatternFill("solid", fgColor="1F3864")
    HF = Font(color="FFFFFF", bold=True)
    COLORS = {"OK": "E2EFDA", "Falla raíz": "F8CBAD", "Afectado por padre (suprimir)": "FFE699",
              "Falla propia": "FCE4D6", "Sin datos": "D9D9D9", "Sin relaciones": "DDEBF7",
              "Debió suprimirse": "FFE699", "Alerta raíz válida": "F8CBAD",
              "Carrera de tiempos (revisar)": "FFD966", "Válida (previa a la caída del padre)": "E2EFDA", "Alta": "F8CBAD",
              "Media": "FFE699", "Baja": "DDEBF7"}

    def sheet(title, rows_, cols, color_col=None, first=False):
        ws = wb.active if first else wb.create_sheet()
        ws.title = title
        ws.append([c[1] for c in cols])
        for c in ws[1]:
            c.fill = HDR; c.font = HF; c.alignment = Alignment(wrap_text=True, vertical="center")
        for r in rows_:
            ws.append([r.get(c[0], "") for c in cols])
            if color_col:
                v = r.get(color_col, "")
                if v in COLORS:
                    idx = [c[0] for c in cols].index(color_col) + 1
                    ws.cell(ws.max_row, idx).fill = PatternFill("solid", fgColor=COLORS[v])
        for i, c in enumerate(cols, 1):
            ws.column_dimensions[get_column_letter(i)].width = c[2] if len(c) > 2 else 18
        ws.freeze_panes = "A2"
        if rows_:
            ws.auto_filter.ref = ws.dimensions
        return ws

    # Resumen
    ws = wb.active; ws.title = "Resumen"
    cl = D["client"]; sc = D["scope"]
    st = Counter(s["estado"] for s in R["state_rows"])
    al = Counter(a["clasificacion"] for a in R["alert_rows"])
    fs = Counter(f["severidad"] for f in R["findings"])
    cfg = Counter(c["severidad"] for c in R["cfg_rows"])
    lines = [("Bricks — Relaciones y estados de componentes", ""),
             ("Cliente", f"{cl.get('id', '')} · {cl.get('name', '')}"),
             ("Alcance", json.dumps(sc, ensure_ascii=False)),
             ("Datos consultados", D.get("generated_at") or ""), ("", ""),
             ("Componentes analizados", len(R["state_rows"])),
             ("Relaciones (aristas)", len(R["rel_rows"])), ("", ""), ("ESTADOS", "")]
    lines += [(k, st.get(k, 0)) for k in ["OK", "Falla raíz", "Afectado por padre (suprimir)",
                                          "Falla propia", "Sin datos", "Sin relaciones"]]
    lines += [("", ""), ("ALERTAS ACTIVAS", "")]
    lines += [(k, al.get(k, 0)) for k in ["Alerta raíz válida", "Debió suprimirse", "Carrera de tiempos (revisar)",
                                          "Válida (previa a la caída del padre)", "Válida (padre sin falla)",
                                          "Sin fecha para comparar (revisar)"]]
    lines += [("Criterio", "Solo 'Debió suprimirse' las alertas de descendientes emitidas DESPUÉS de la caída de "
                           f"comunicación del padre. Previas = normales (se omiten). 'Carrera' = emitidas hasta "
                           f"{D.get('ventana_carrera_s', 120)} s antes de que el padre registrara la caída.")]
    if D.get("freshness_truncated"):
        lines += [("Aviso", "La frescura del sitio vino truncada (get_current_state limita la lista); "
                            "'Sin datos' puede estar incompleto")]
    lines += [("", ""), ("HALLAZGOS DE TOPOLOGÍA", "")] + [(k, fs.get(k, 0)) for k in ["Alta", "Media", "Baja", "Info"]]
    lines += [("", ""), ("CONFIGURACIÓN DE CASCADA (reglas de hijos)", "")] + \
             [(k, cfg.get(k, 0)) for k in ["Alta", "Media", "Info", "OK"]]
    lines += [("", ""), ("ETIQUETAS ENCONTRADAS (lectura: origen → etiqueta → destino)", "")]
    for lab, n in R["labels_found"].most_common():
        s = R["sem"].get(lab)
        if s:
            lines.append((f"{lab} ({n})", s["frase"].format(a="ORIGEN", b="DESTINO") +
                          f" · padre = {'DESTINO' if s['parent_end'] == 'to' else 'ORIGEN'}"
                          f" · {s['kind']} · {'propaga supresión' if s['propagates'] else 'NO propaga supresión'}"))
        else:
            lines.append((f"{lab} ({n})", "NO CATALOGADA — confirmar semántica"))
    for a, b in lines:
        ws.append([a, b])
    ws["A1"].font = Font(bold=True, size=14)
    for r in ws.iter_rows():
        if r[0].value and str(r[0].value).isupper():
            r[0].font = Font(bold=True)
    ws.column_dimensions["A"].width = 48; ws.column_dimensions["B"].width = 90

    sheet("Estados", R["state_rows"], [
        ("id", "ID", 9), ("componente", "Componente", 42), ("tipo", "Tipo", 13), ("sitio", "Sitio", 8),
        ("estado", "Estado", 26), ("raiz_falla", "Raíz de la falla", 34), ("padre", "Padre (comunicación)", 34),
        ("nivel", "Nivel", 7), ("hijos_directos", "Hijos directos", 9), ("descendientes", "Descendientes", 11),
        ("falla_comunicacion", "Falla comunicación", 11), ("evidencia_falla", "Evidencia", 40),
        ("frescura", "Frescura", 9), ("ultimo_dato", "Último dato", 22), ("alertas_activas", "Alertas activas", 9),
        ("cadena", "Cadena hasta la raíz", 90)], color_col="estado")
    sheet("Alertas", R["alert_rows"], [
        ("event_id", "Evento", 11), ("componente", "Componente", 40), ("falla", "Falla", 46),
        ("severidad", "Severidad", 10), ("activada", "Activada", 19), ("es_comunicacion", "Pérdida com.", 10),
        ("clasificacion", "Clasificación", 26), ("raiz", "Padre en falla (raíz)", 34), ("momento", "Momento", 60),
        ("regla_valida_padre", "Regla valida padre", 10), ("rule_id", "Regla", 8), ("componente_id", "ID comp.", 9)],
        color_col="clasificacion")
    sheet("Jerarquía", R["tree_rows"], [("arbol", "Árbol de comunicación", 80), ("nivel", "Nivel", 7),
                                       ("id", "ID", 9), ("tipo", "Tipo", 14), ("estado", "Estado", 26)],
          color_col="estado")
    if R["other_tree"]:
        sheet("Otras jerarquías", R["other_tree"], [("jerarquia", "Jerarquía", 14), ("arbol", "Árbol", 70),
                                                   ("nivel", "Nivel", 7), ("id", "ID", 9)])
    sheet("Hallazgos topología", sorted(R["findings"], key=lambda f: ["Alta", "Media", "Baja", "Info"].index(f["severidad"])
                                         if f["severidad"] in ["Alta", "Media", "Baja", "Info"] else 9), [
        ("severidad", "Severidad", 10), ("tipo", "Tipo", 28), ("componente_id", "ID", 9),
        ("componente", "Componente", 40), ("detalle", "Detalle", 100)], color_col="severidad")
    sheet("Config cascada", R["cfg_rows"], [
        ("severidad", "Severidad", 10), ("rule_id", "Regla", 8), ("regla", "Nombre", 50),
        ("tipo_hijo", "Tipo hijo", 22), ("tipos_padre", "Tipo(s) padre", 26), ("es_comunicacion", "Pérdida com.", 10),
        ("genera_alerta", "Genera alerta", 9),
        ("discon_parent_validation", "Valida padre", 9), ("parent_validation", "parent_validation", 30),
        ("cubre_padre_con", "Cubre padre con", 18), ("hallazgo", "Hallazgo", 90)], color_col="severidad")
    sheet("Reglas", R["rule_rows"], [("rule_id", "Regla", 8), ("regla", "Nombre", 52), ("tipo", "Tipo", 18),
                                    ("tipo_componente", "Tipo componente", 24), ("codigos", "Códigos", 30),
                                    ("es_comunicacion", "Pérdida com.", 10), ("motivo", "Motivo", 55)])
    sheet("Relaciones", R["rel_rows"], [
        ("arista", "Arista", 8), ("frase", "Lectura (frase)", 80), ("label", "Etiqueta", 16), ("tipo", "Tipo", 13),
        ("padre", "Padre", 40), ("hijo", "Hijo", 40), ("propaga", "Propaga supresión", 10),
        ("from", "Origen", 9), ("to", "Destino", 9)])
    wb.save(path)


def summary(R):
    st = Counter(s["estado"] for s in R["state_rows"])
    al = Counter(a["clasificacion"] for a in R["alert_rows"])
    lab = R["ctx"]["label_of"]
    roots = [s for s in R["state_rows"] if s["estado"] == "Falla raíz"]
    return {
        "estados": dict(st), "alertas": dict(al),
        "etiquetas": dict(R["labels_found"]),
        "fallas_raiz": [{"id": s["id"], "componente": s["componente"], "descendientes": s["descendientes"],
                         "evidencia": s["evidencia_falla"]} for s in roots],
        "debieron_suprimirse": [{"evento": a["event_id"], "componente": a["componente"], "falla": a["falla"],
                                 "raiz": a["raiz"], "momento": a["momento"]}
                                for a in R["alert_rows"] if a["clasificacion"] == "Debió suprimirse"][:60],
        "carrera_de_tiempos": [{"evento": a["event_id"], "componente": a["componente"], "falla": a["falla"],
                                "raiz": a["raiz"], "momento": a["momento"]}
                               for a in R["alert_rows"] if a["clasificacion"].startswith("Carrera")][:60],
        "hallazgos_por_tipo": {f"{k[0]} · {k[1]}": v for k, v in
                               Counter((f["severidad"], f["tipo"]) for f in R["findings"]).most_common()},
        "cascada_hallazgos": [c for c in R["cfg_rows"] if c["severidad"] in ("Alta", "Media")],
    }


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("entrada")
    ap.add_argument("-o", "--salida")
    ap.add_argument("--simular", type=int, help="site_component_id que se simula en falla de comunicación")
    ap.add_argument("--componente", type=int, help="ver relaciones/estado de un componente")
    ap.add_argument("--ventana-carrera", type=int, default=120,
                    help="segundos antes de la caída del padre en que una alerta hija se marca como carrera de tiempos")
    a = ap.parse_args()
    D = load(a.entrada)
    D["ventana_carrera_s"] = a.ventana_carrera
    R = analyze(D)
    if a.simular:
        print(json.dumps(simulate(R, a.simular), ensure_ascii=False, indent=1))
    if a.componente:
        print(json.dumps(component_view(R, a.componente), ensure_ascii=False, indent=1, default=str))
    if a.salida:
        write_excel(R, D, a.salida)
        print(json.dumps(summary(R), ensure_ascii=False, indent=1, default=str))
    if not (a.simular or a.componente or a.salida):
        print(json.dumps(summary(R), ensure_ascii=False, indent=1, default=str))


if __name__ == "__main__":
    main()
```

## Anexo B — Formato de entrada

Es un JSON. Cada lista admite filas compactas (listas, en el orden indicado) o dicts con esos mismos nombres de campo. Solo hacen falta las aristas; todo lo demás es opcional, pero sin alertas no hay clasificación y sin inventario no hay auditoría de cascada.

```json
{
  "client": {"id": 62, "name": "Ladrillera Santafe"},
  "scope": {"type": "site", "ids": [1324]},
  "generated_at": "2026-11-03T22:27:00Z",

  "edges_by_target": {"sendsDataTo": {"32101": [32313, 32314], "37586": [32101, 34556]}},
  "edges_by_source": {"hasSubMeter": {"34115": [34119, 34120]}},
  "edges": [[821, 31106, 36798, "sendsDataTo"]],

  "vertices":   [[32101, "Soacha DVR - Monserrate", "DVR", "CCTV", 1324]],
  "components": [[32101, 1324, 15, "DVR inhouse", "Soacha DVR - Monserrate"]],
  "freshness":  [[32105, "stale", "2026-11-03T14:09:27Z"]],
  "alerts":     [[10109864, 37586, 1282, "EUT - Switch Desconectado (Cascada)", "critical", "2026-10-26T16:00:07Z", "active", "2026-10-26T16:00:06Z", "2026-10-26T16:00:05Z"]],
  "pending":    [[10109000, 37586, 1292, "EUT - Switch Desconectado (Cascada - Sombra)", "2026-10-26T15:59:00Z"]],
  "rules":      [[1282, "EUT - Switch Desconectado (Cascada)", "disconnection", 151, 1, "257,1051,239,1266,1292", "unreachable", "active", 0]],

  "alert_rule_ids": [1282, 1283, 1289],
  "freshness_truncated": false,
  "comm_rule_ids": [],
  "not_comm_rule_ids": [],
  "label_semantics": {"usesChannel": {"parent_end": "to", "kind": "comunicacion", "propagates": true, "frase": "{a} usa el canal de {b}"}}
}
```

| Lista | Orden de campos | Fuente MCP |
|---|---|---|
| `edges` | id, from, to, label | `list_brick_edges` (`from_vertex_id`, `to_vertex_id`, `label`) |
| `edges_by_target` | `{label: {to: [from…]}}` | Igual, agrupado: úsalo para `sendsDataTo` (muchos hijos por padre) |
| `edges_by_source` | `{label: {from: [to…]}}` | Igual, agrupado: úsalo para `hasSubMeter` |
| `vertices` | id, label, source_type, system, site_id | `list_brick_vertices` |
| `components` | id, site_id, type_id, type_name, description | `get_site_components` (`id`, `component_id`, `component_name`, `description`) |
| `freshness` | id, status, last_seen | `get_current_state(...).data_freshness` |
| `alerts` | event_id, site_component_id, rule_id, falla, severity, activated_at, status, created_at, event_at | `get_alerts` (`id`, `site_component_id`, `rule_id`, `falla`, `severity`, `activated_at_utc`, `status`, `created_at_utc`, `event_at_utc`) |
| `pending` | event_id, site_component_id, rule_id, falla, created_at | `get_events(statuses="pending", rule_ids=…)` |
| `rules` | id, name, type, type_component_id, discon_parent_validation, parent_validation, disconnection_codes, status, activate_on | `get_site_rules` (`component_id` es el tipo) |

Notas:
- Las fechas deben ir todas en la misma zona: lo preferible son los campos `*_utc` de Reveal. El script las parsea (ISO o `YYYY-MM-DD HH:MM:SS`) y clasifica cada alerta hija por tiempo: después de la caída del padre (debió suprimirse), hasta 120 s antes (carrera de tiempos) o antes (previa, se omite). Sin `created_at`/`event_at` se usa `activated_at`.
- `alert_rule_ids`: los `rule_id` que `get_rule_classifications` marca como alerta. Si no lo pasas, la columna «Genera alerta» queda en «¿?».
- `freshness_truncated`: ponlo en `true` si `get_current_state` truncó `data_freshness`.
- Si varios sitios repiten reglas, basta con una fila por `rule_id`.
- `parent_end`: `"to"` si el padre es el destino de la frase y `"from"` si es el origen. `propagates: true` solo para dependencias de comunicación.
- Las aristas pueden cruzar sitios (por ejemplo, el appliance de otra sede). Inclúyelas para que la cadena llegue a la raíz.
