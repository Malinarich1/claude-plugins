---
name: trelloutlook
description: Kanban TrellOutlook (SIM, Vitrify, Cubicaciones, TrellOutlook) por su API REST. Usala cuando te pidan trabajar o tomar una tarea o tarjeta del tablero, panel o kanban, moverla a EN PROCESO o PRUEBA INTERNA, comentar el avance, anotar un hallazgo, ver qué hay pendiente o abierto, o dejar y correr dev-prompts del front o del back.
---

# TrellOutlook — runbook del agente

Sos el agente de un humano (Nicolás: back / Vicente: front). Todo lo que crees, muevas o comentes queda **atribuido a él**, con su nombre y su cara: escribí como si él lo fuera a leer, porque lo va a leer. No arrancás nada por tu cuenta: **la tarea la elige el humano**.

## Antes de la primera llamada

1. `export PYTHONIOENCODING=utf-8 MSYS_NO_PATHCONV=1` **siempre, primero**. Sin la primera, imprimir el board revienta con `UnicodeEncodeError` (cp932) al llegar a la `Ó` de REVISIÓN. Sin la segunda, en Git Bash `/auth/me` se convierte en `C:/Program Files/Git/auth/me` y urllib tira `InvalidURL`: **entrecomillar el path no evita nada**, solo la variable lo frena. Y ojo, las llamadas con `?` (`/board?board_id=…`) sobreviven igual: por eso el error parece intermitente y terminás descartando el shell como causa justo cuando te falla `/auth/me` o el detalle de una tarjeta. (Efecto lateral: con `MSYS_NO_PATHCONV=1`, `curl -o /dev/null` deja de andar — usá `-o nul`.) **En PowerShell** eso es `$env:PYTHONIOENCODING="utf-8"` — `export` no existe y te tira "The term 'export' is not recognized"; `MSYS_NO_PATHCONV` no hace falta, ahí no hay conversión de rutas.
2. El PAT sale de `$TRELLOUTLOOK_PAT` (settings.json del humano). **Nunca** lo escribas en un archivo del repo, en un commit ni en un comentario de tarjeta: los comentarios los ven los clientes del tablero.
3. Base: `https://lectorcorreotrello2-backend.fly.dev/api/v1`, header `Authorization: Bearer <PAT>`. (El `X-Panel-Token` del token de agente legacy todavía responde 200, pero está retirado: no lo uses, el día que se caiga toda la skill da 401 y la glosa de errores te manda a perseguir un PAT que está perfecto.)
4. `GET /auth/me` → confirma `agent_token: true` y con qué identidad estás firmando. El `name` te dice **de qué lado estás**: Nicolás = `back`, Vicente = `front` (los valores son literales, el backend rechaza cualquier otro con 400). En la bandeja el lado se lee en dos direcciones: cuando **vos dejás** un prompt, `metadata.prompt_para` es el **otro** lado; cuando **buscás tu turno**, filtrás `metadata.prompt_para` = **tu** lado.

`curl -d '{"body":"..."}'` inline **destruye lo no-ASCII en silencio**: verificado sobre el cable, `revisión ñ ✅` sale como `revision n ?`. No falla y no avisa, y el comentario queda firmado por tu humano y escrito como por un analfabeto. Los bodies con texto van en un **archivo .json en UTF-8** (`--data-binary @body.json`) o, mejor, por este helper. Los heredocs largos en bash a veces fallan con `unexpected EOF` (pasa con los de muchas líneas y comillas anidadas): ante la duda, escribí el archivo y ejecutalo.

```python
# tl.py   python tl.py GET "/tasks/<id>"   |   python tl.py POST "/tasks/<id>/comments" body.json
#         OUT=t.json python tl.py GET "/tasks?board_id=<id>"   -> guarda crudo a archivo, no imprime
import json, os, sys, urllib.request, urllib.error
BASE = "https://lectorcorreotrello2-backend.fly.dev/api/v1"
def call(method, path, body=None):
    data = json.dumps(body, ensure_ascii=False).encode("utf-8") if body is not None else None
    req = urllib.request.Request(BASE + path, data=data, method=method, headers={
        "Authorization": "Bearer " + os.environ["TRELLOUTLOOK_PAT"],
        "Content-Type": "application/json; charset=utf-8"})
    try:
        with urllib.request.urlopen(req) as f:
            return f.status, f.read().decode("utf-8")
    except urllib.error.HTTPError as e:
        return e.code, e.read().decode("utf-8")   # el cuerpo del error dice qué arreglar
def get(path):                                    # JSON parseado, para filtrar desde python
    st, raw = call("GET", path)
    d = json.loads(raw or "null")
    if st != 200: raise SystemExit((st, d))       # 404 = no existe O no lo ves
    return d
if __name__ == "__main__":
    b = json.load(open(sys.argv[3], encoding="utf-8")) if len(sys.argv) > 3 else None
    st, raw = call(sys.argv[1], sys.argv[2], b)
    print(st)
    if os.environ.get("OUT"):
        open(os.environ["OUT"], "w", encoding="utf-8").write(raw)
        print(len(raw), "bytes ->", os.environ["OUT"]); sys.exit()
    try: d = json.loads(raw or "null")
    except ValueError: print(raw[:4000]); sys.exit()    # /contract y /guia son markdown
    if isinstance(d, dict):
        d.pop("description_rich", None); d.pop("creator", None)
        if len(d.get("description") or "") > 1200:
            d["description"] = d["description"][:1200] + " …(cortada: OUT= para leerla entera)"
        if d.get("events"): d["events"] = d["events"][-2:]
    print(json.dumps(d, ensure_ascii=False, indent=1)[:4000])
```

Los dos recortes de la impresión no son cosméticos. El detalle de una tarjeta ya trabajada pesa ~12 KB: la `description` sola (7 KB en una tarjeta real) y el historial de `events` te empujan `metadata`, `assignees`, `attachments` y `branches` fuera del corte de 4000, o sea leés media descripción y no ves ni los responsables, ni las capturas, ni las ramas — justo lo que "Trabajá la tarea X" te manda leer. Con el recorte entran `metadata`, `assignees` y `attachments`, y te quedan los últimos 2 eventos, que es lo que necesitás para saber quién la tocó. `branches` es la **última** clave del detalle y en una tarjeta gorda se corta igual: si te importan las ramas, pedí la tarjeta con `OUT=`. Y el `pop` del doc rico **no es hipotético**: hoy más de la mitad de las tarjetas del panel son `description_format: rich` (56 de 159, medido). Por eso, en una tarjeta con contenido humano **no mandes `description` sola en un PATCH**: eso descarta el doc rico y sus imágenes embebidas. Tu avance va en un comentario.

### Regla dura de contexto

**Nunca dejes que una respuesta de lista llegue cruda al contexto.** `GET /board?board_id=<TrellOutlook>` pesa 443 KB ≈ **110k tokens**, y `/tasks?board_id=` del mismo tablero, 421 KB. No hay recorte del lado del servidor: `limit`, `fields` y `column_id` **no existen** y `?q=` **no busca** (responde 200, ignora el texto y devuelve todo). El `[:4000]` del helper es lo que te salva ~100k tokens por llamada exploratoria; no lo borres. (`/tasks` sí tiene un tope duro de 500 filas del lado del servidor, pero es tan alto que no te protege de nada: hoy los cuatro tableros juntos dan 159.)

Cuando necesites la lista entera, bajala a archivo y filtrala en python **abriendo con `encoding="utf-8"`**:

```
OUT=t.json python tl.py GET "/tasks?board_id=<id>"
json.load(open("t.json", encoding="utf-8"))     # PYTHONIOENCODING arregla el print, NO el open
```

Sin ese `encoding` explícito el `json.load` muere con `UnicodeDecodeError` cp932 y parece que el archivo se bajó corrupto. Los tableros de proyecto son chicos (SIM ~13 KB): el número grande es el del tablero TrellOutlook.

- `GET /contract` (~13k tokens) es la fuente de verdad de la API, siempre al día (`X-Contract-Sha`): leelo cuando necesites la forma exacta de un request en vez de suponerla. `GET /guia` (~2.3k) es la fuente canónica y versionada del **protocolo de la bandeja** (`X-Guia-Sha`): pedila antes de tocar un dev-prompt, no reconstruyas el protocolo de acá.
- Los dos son **markdown, no JSON**: bajalos a archivo (`OUT=guia.md python tl.py GET "/guia"`) y leé el archivo. `get()` sobre ellos tira `JSONDecodeError`.
- `GET /version` → `{git_sha, contract_sha, guia_sha}`.

## Tableros y columnas

SIM `09b80cf9-6ff1-4acb-81a8-c217746763ee` (carpeta SALFACORP) · TrellOutlook `0ffd3d58-f8bc-4bfc-a8be-770836fee216` · Vitrify `eb9e855b-d6f1-461d-991d-1678e77ccc96` · Cubicaciones `5ff265f1-c6e1-48cc-a35b-a6f98f215745`.

**Las columnas cambian por tablero: descubrilas siempre — y el mapa sale de `GET /boards`, que pesa 6 KB.** Ahí viene cada tablero visible con su `columns_summary` (`{id, name, kind, tasks_count}` de **todas** sus columnas) y sus `repos`. Pedí `GET /board?board_id=` (443 KB) **solo cuando necesites las tareas**, y a archivo.

```python
from tl import get              # tl.py en el cwd, con TRELLOUTLOOK_PAT exportado
BID = "0ffd3d58-…"              # el tablero del proyecto en el que estás trabajando
b = next(x for x in get("/boards")["boards"] if x["id"] == BID)
cols = {c["name"]: c["id"] for c in b["columns_summary"]}
ABIERTO    = next(c["id"] for c in b["columns_summary"] if c["kind"] == "abierto")
EN_PROCESO = cols["EN PROCESO"]
PRUEBA     = cols["PRUEBA INTERNA"]
```

- ABIERTO se resuelve por `kind == "abierto"` (hay exactamente una por tablero): es el único ancla que no depende del nombre. (En `GET /board` el mismo ancla se llama `is_default_inbox`.)
- **Para el resto el `kind` no distingue nada**: `en_curso` son dos en SIM/Vitrify/Cubicaciones (`EN PROCESO`, `PRUEBA INTERNA`) y **tres** en TrellOutlook, porque `Pulir` también es `en_curso` y es territorio humano. `revision` son dos en los cuatro. Mirá el **nombre**.
- **Nunca matchees nombres de columna por substring ni `startswith`**: `REVISIÓN CLIENTE OK` empieza con `REVISI…`. Igualdad exacta. Los nombres varían entre tableros (`REVISIÓN CLIENTE FAIL` en TrellOutlook, `REVISIÓN CLIENTE NO OK` entero en los otros tres). Esta regla es para columnas; para buscar tarjetas por texto, mirá "Trabajá la tarea X".
- Toda columna que no sea ABIERTO / EN PROCESO / PRUEBA INTERNA es **territorio humano**.

## Quién mueve qué — la regla que no se rompe

| Columna | La mueve |
|---|---|
| ABIERTO | el humano, o **vos** (hallazgo nuevo / devolución a la otra parte) |
| EN PROCESO | **vos**, al arrancar. Verla ahí ya avisa que alguien la está haciendo |
| PRUEBA INTERNA | **vos**, al terminar. **Es tu estado terminal: nunca pasás de acá** |
| REVISIÓN y todo lo que sigue | **solo humanos** — sacarla de ahí hacia EN PROCESO, solo si tu humano te lo pide |

REVISIÓN es la puerta al **cliente final**, no una revisión interna: un humano prueba primero. No hay claim para tarjetas comunes (el claim existe solo para los dev-prompts).

## Endpoints

```
GET   /boards                  árbol: tableros con columns_summary y repos (6 KB)
GET   /board?board_id=<id>     tablero entero; las tareas van en columns[].tasks[] (GRANDE: a archivo)
GET   /tasks?board_id=<a>,<b>  tareas planas de uno o varios tableros (coma), con board_name,
                               column_name, column_kind, description y metadata
GET   /tasks/{id}              detalle: description, metadata, assignees, attachments[], branches[], events
POST  /tasks                   {board_id, column_id, title, description, metadata}
PATCH /tasks/{id}              {title?, description?, metadata?}  metadata = shallow-merge
POST  /tasks/{id}/move         {"column_id": "..."} mismo tablero; after_task_id opcional
GET|POST /tasks/{id}/comments  {"body": "..."} texto plano, máx 10.000 chars
POST  /tasks/{id}/branches     {"repo_id": "...", "branch": "..."} repo de ESE tablero
PUT   /tasks/{id}/assignees    {"user_ids":[...]} REEMPLAZA la lista entera
GET   /users?board_id=<id>     únicos ids válidos como responsable de ese tablero
```

- `metadata` hace **shallow-merge** en PATCH: mandar `{"area":"FRONT"}` conserva `prioridad` y el resto. En uso: `prioridad` (`ALTA|MEDIA|NORMAL`), `area` (`BACK|FRONT|BACK-FRONT|SAP`). **Son convención, no enum**: el backend guarda `metadata` tal cual y un valor mal escrito entra con 200 y deja una tarjeta que el panel no sabe pintar. Copialos exactos.
- `move` **no cruza tableros**: con una columna de otro tablero responde un **`404` genérico**, no un 400 explicativo. Ante un 404 en `move`, mirá primero de qué tablero es la columna antes de sospechar de permisos.
- La lista plana es liviana: `assignees` y `comments_count` vienen vacíos, y los `attachments` (capturas que dejó el humano) solo aparecen en el detalle. Al revés, el detalle trae `column_id` pero **no** `column_name`. `?assignee=<uuid>` sí filtra.

---

# Situaciones

## "Trabajá la tarea X"

Si te la nombran por título y no por id: `OUT=t.json python tl.py GET "/tasks?board_id=<id>"` y filtrá por **substring en minúsculas sobre `title` y `description`** — la lista plana trae los dos y muchas veces el tema está solo en la descripción. (La igualdad exacta es la regla de los nombres de columna, no la de buscar tarjetas.)

`GET /tasks/{id}` (leé `description`, `metadata`, `assignees`, `attachments`, `branches`) + `GET /tasks/{id}/comments` para el hilo. Como el detalle no trae el nombre de la columna, resolvé su `column_id` contra el mapa que ya armaste **antes de mover**:

- **En ABIERTO** → `POST /tasks/{id}/move` a `EN_PROCESO` antes de tocar código.
- **Ya en EN PROCESO** → no la muevas: el move a la misma columna sale 200, la sube al tope y les avisa a los humanos un arranque falso. Alguien la está haciendo: leé el último comentario y `events`, y confirmá con tu humano que la seguís vos.
- **En REVISIÓN o más allá** → no la arrastres por tu cuenta. Si tu humano te la pide explícitamente (el caso típico: rechazo del cliente), traela a EN PROCESO: lo prohibido es el **destino** REVISIÓN y más allá, no el origen.

No toques `assignees` al arrancar: el move a EN PROCESO ya es la señal, y el PUT reemplaza la lista entera (podés borrar a un responsable real que no es ninguno de tus dos humanos).

## "Terminé"

Comentario con qué hiciste, qué archivos tocaste, **cómo verificarlo**, **dónde quedó** (rama / PR; si deployaste, el `git_sha` de `GET /version`) y qué quedó afuera → `move` a `PRUEBA INTERNA` → **ahí frenás** y le avisás al humano que quedó para que pruebe.

Si el proyecto tiene repos cargados (vienen en `repos` de `GET /boards`), registrá la rama: `POST /tasks/{id}/branches {"repo_id":"…","branch":"…"}`, con un repo de **ese** tablero. Una rama por repo y tarea: la segunda da 409 y se edita con `PATCH /tasks/{id}/branches/{branchID}`. Si el proyecto no tiene repos, la rama va en el comentario.

Antes de mover, mirá `metadata.area`: si es `BACK-FRONT` y solo hiciste tu mitad, PRUEBA INTERNA es mentira. Comentás lo tuyo y hacés la devolución completa: `move` a **ABIERTO** (la tarjeta está en EN PROCESO, la moviste vos al arrancar — no se queda sola) + `PATCH {"metadata":{"area":"FRONT"}}` + dev-prompt con `metadata.prompt_tarea_id` = el id de esa tarjeta; es la receta de "Me trabé: depende del front / del back". PRUEBA INTERNA es cuando la tarjeta **entera** está lista para que la prueben.

El avance va **en un comentario**, no reescribiendo la descripción: el pedido original del humano queda intacto. `description` **no** hace shallow-merge, un PATCH la reemplaza entera. Solo la tocás si está **vacía**: `PATCH {"description":"..."}` en texto plano (no armes `description_rich`). Si está **a medias pero con contenido del humano, no la patchees**: mandar solo `description` descarta el doc rico y libera sus imágenes embebidas, o sea le borrás el formato y las capturas. Tu complemento va en un comentario.

## "Me trabé: necesito algo del humano"

Comentás exactamente qué te frena y qué necesitás para seguir. **Dejás la tarjeta en EN PROCESO** (sigue siendo tuya).

## "Me trabé: depende del front / del back"

Comentás qué falta → `move` a **ABIERTO** con las instrucciones → `PATCH {"metadata":{"area":"FRONT"}}` → dev-prompt para el agente de esa parte **con `metadata.prompt_tarea_id` = el id de esa misma tarjeta**. Si lo omitís, el backend crea una tarjeta de trabajo nueva: te quedan dos para lo mismo y el hilo partido.

## "Encontré un bug o deuda al pasar"

No lo arregles de prepo. Tarjeta nueva en ABIERTO del **mismo proyecto** (qué pasa, dónde, cómo reproducirlo, que salió trabajando la tarjeta X) y seguís con lo tuyo. **Mandá siempre `board_id`**: sin él la tarea cae en el tablero principal, o sea en el proyecto equivocado.

## "¿Qué hay pendiente?"

`GET /tasks?board_id=<a>,<b>` (varios ids separados por coma en **una sola llamada**; cada tarea trae `board_name`) → filtrás `column_name` = ABIERTO **y tirás las que tengan `metadata.tipo == "dev-prompt"`**. El kanban las oculta a propósito (no entran en `columns_summary`): son coordinación entre agentes, no trabajo pendiente. Sin ese descarte le reportás como pendiente la coordinación entre agentes —hoy son la mayoría del ABIERTO de TrellOutlook— y encima algún prompt repite el título de su tarjeta enlazada: tu resumen no coincide con lo que él ve en pantalla. Resumís y podés sugerir por dónde empezar; **no arranques**.

## Bandeja de dev-prompts (front ↔ back)

Un dev-prompt es una **tarea** con `metadata.tipo: "dev-prompt"`. **Hoy todos viven en el tablero principal (TrellOutlook)** y el proyecto del que hablan va en `metadata.prompt_proyecto`, que es lo que dice `/guia` y lo que hace el panel: los 53 que existen están ahí. Está decidido que en el futuro cada uno viva en el tablero de su proyecto, pero eso espera a que exista mover tareas entre tableros y a que `/guia` lo diga: **hasta entonces, creálos en el principal**, o el otro agente no los va a encontrar.

**Todo lo de la bandeja va adentro de `metadata`** (que hace shallow-merge): `prompt_para`, `prompt_estado`, `prompt_autor`, `prompt_proyecto`, `prompt_tarea_id`, `prompt_respuesta`, `prompt_respuesta_tipo`, `prompt_prioridad`, `prompt_padre`. Ninguno existe en la raíz de la tarea. El PATCH con los campos sueltos responde **200 sin cambiar nada** (el handler solo lee `title`, `description`, `metadata`), así que creés que respondiste y el prompt sigue pendiente con el claim tomado. Y el POST **solo te frena con 400 si `metadata` trae `tipo: "dev-prompt"`**: si mandás todo suelto y sin `metadata`, responde **201** y te deja una tarjeta común que nadie va a leer como prompt.

La bandeja **también la dispara el humano**: podés listar los prompts de tu lado y avisar qué hay, pero correrlos te los pide él. El protocolo completo —campos, enums, estados, hilos— está en `GET /guia`.

- **Ver tu turno:** `OUT=p.json python tl.py GET "/tasks?board_id=<id1>,<id2>,<id3>"` (una sola llamada, con `board_name`) y sobre los `metadata.tipo=="dev-prompt"` filtrás dos cosas: **(a)** `metadata.prompt_estado=="pendiente"` con `metadata.prompt_para` = **tu** lado, que son los que te piden; **(b)** los que **pediste vos** (`prompt_para` = el otro lado) que volvieron `"corrido"` con `metadata.prompt_respuesta_tipo == "requiere"`: leés `prompt_respuesta`, hacés lo que pida y cerrás con `PATCH {"metadata":{"prompt_estado":"cerrado"}}`. Los `"cierra"` se cierran solos, esos no son tu turno. Pedir los cuatro tableros de una es una precaución barata, no una corrección: como hoy los prompts viven todos en el principal, la receta de `/guia` (que consulta solo ese) devuelve lo mismo. El día que se repartan por proyecto, esta receta sigue andando sin tocarla.
- **Dejar uno:** `POST /tasks` con **`board_id` del tablero principal** (ahí viven todos hoy) y `metadata` = `{"tipo":"dev-prompt","prompt_para":"front","prompt_estado":"pendiente","prompt_proyecto":"…"}`, más una descripción con la forma exacta de lo que necesitás (endpoint, campos, ejemplo) y qué ya está hecho de tu lado. `prompt_proyecto` **etiqueta, no ubica**: el ejemplo de `/guia` omite `board_id` y sin él el prompt y su tarjeta caen en el tablero principal. Sin `column_id` nace en el ABIERTO de ese tablero.
- **`metadata.prompt_tarea_id`:** si el prompt sale de una tarjeta que **ya existe** (el caso de arriba: la que devolviste a ABIERTO), mandalo con el id de esa tarjeta. Omitirlo dispara el autocreado del backend, que es solo para pedidos nuevos que todavía no tienen tarjeta — y en ese caso **no la crees vos aparte**, la crea el backend.
- **Hilo de una tarjeta:** `GET /tasks/{id}/prompts` va con el id de la **tarjeta de trabajo**, no el del prompt. Con el del prompt devuelve `[]` y 200, y concluís mal que no hay hilo.
- **Tomar:** `POST /tasks/{id}/claim` (acepta `{"agente":"..."}`, lease 45 min). `409` = ya lo tiene otro. `404` = esa tarea no es un dev-prompt. `DELETE .../claim` lo suelta.
- **Responder:** `PATCH {"metadata":{"prompt_estado":"corrido","prompt_respuesta":"<el texto>","prompt_respuesta_tipo":"cierra"}}`. Con `"cierra"` el backend lo pasa a `cerrado` y lo archiva; con `"requiere"` queda esperando a la otra parte (y si el autor fue tu lado, vuelve a tu turno por la vía (b)). Ese PATCH libera el claim solo. `cerrado` es el estado del **prompt**, no de la tarjeta: la tarjeta de trabajo enlazada sigue el flujo normal (EN PROCESO al arrancar, PRUEBA INTERNA al terminar) y frena ahí igual.
- Si no estás de acuerdo con una respuesta, no reabras: prompt nuevo con `metadata.prompt_padre` = el id del anterior.
- Al decir "listo, deployado", citá el `git_sha` de `GET /version`. Quien verifica compara contra `/version` **antes** de smokear: si no coincide, todavía no está deployado (no es un bug).

## Ajustes que SÍ podés hacer

- **Prioridad**: `PATCH {"metadata":{"prioridad":"ALTA"}}`.
- **Responsables**: `PUT /tasks/{id}/assignees` **reemplaza** la lista entera — leé los actuales del detalle y mandá la lista final. Los ids válidos salen de `GET /users?board_id=<el del proyecto>`; uno sin acceso al proyecto da 400.

Siempre explicás el porqué en el comentario.

## Errores

- `400` body inválido o valor fuera de enum — el mensaje dice qué arreglar, leelo. Ojo: los únicos enums que el backend valida son los `prompt_*` de la bandeja; `metadata` de una tarjeta común se guarda tal cual.
- `401` falta o no sirve el token — **excepto `/auth/tokens`, que responde 401 aunque tu PAT esté perfecto**: acuñar o listar tokens exige la sesión del humano en el panel. El PAT tampoco sirve para el WebSocket (401 por diseño): todo por REST, y para ver cambios volvés a pedir.
- `403` rol insuficiente: un PAT no gestiona miembros, visibilidad, dueños ni roles. Eso lo hace el humano en sesión.
- `404` **"no existe O no lo ves"** — deliberado, no es un bug. Preguntate si ese tablero o tarea es privado para tu humano; y si fue un `move`, si la columna era de otro tablero.
- `409` conflicto de estado (claim ya tomado, columna con tareas, segunda rama del mismo repo).

## Lo que no hacés nunca

- Mover algo **a** REVISIÓN o más allá.
- Borrar tarjetas, columnas o comentarios ajenos.
- Arrancar una tarea que el humano no te pidió.
- Escribir el PAT en cualquier lado.
- Pisar la descripción original de un humano.
