---
name: trelloutlook
description: Kanban TrellOutlook, por sus herramientas MCP o por su API REST. Usala cuando te pidan trabajar o tomar una tarea o tarjeta del tablero, panel o kanban, moverla a EN PROCESO o PRUEBA INTERNA, comentar el avance, anotar un hallazgo, ver qué hay pendiente o abierto, o dejar y correr dev-prompts para la otra parte del equipo.
---

# TrellOutlook — runbook del agente

Sos el agente de una persona del equipo. Todo lo que crees, muevas o comentes queda **atribuido a ella**, con su nombre y su cara: escribí como si lo fuera a leer, porque lo va a leer. No arrancás nada por tu cuenta: **la tarea la elige tu humano**.

Nada de este documento tiene nombres de proyecto ni de persona: los tableros, las columnas, la gente y tu propia identidad se **descubren** en las primeras llamadas. Si alguna vez tenés que escribir un id a mano, es que te salteaste un paso.

## Con qué trabajás

Dos caminos, y **no son equivalentes**.

**Las herramientas `trelloutlook`**, si aparecen en esta sesión. Son seis y hacen el flujo, no la
API: `buscar_tareas`, `ver_tarea`, `mover_tarea`, `comentar_tarea`, `crear_tarea`, `atender_prompt`.
Sus descripciones ya te llegaron con el catálogo, así que acá no se repiten; lo que este documento
agrega es **cuál agarrar en cada situación** y el criterio que ninguna herramienta transmite sola.
Tres cosas que conviene saber antes de usarlas:

- **Hacen el paso completo, no el verbo suelto.** `mover_tarea` mueve *y* deja el comentario en una
  llamada; con `accion="devolver"` además ajusta el área y crea el dev-prompt para la otra parte.
  `crear_tarea` con `pedido` crea la tarjeta y su prompt de forma atómica (quedan las dos o ninguna).
  `atender_prompt` reserva, responde y archiva. No las combines con REST para el mismo paso: te
  quedan dos comentarios y un evento duplicado.
- **Son mucho más baratas.** `buscar_tareas` resuelve en unos cientos de tokens lo que por REST
  cuesta bajarte un tablero entero. Si tenés las herramientas, la "Regla dura de contexto" de más
  abajo casi no te aplica.
- **No pueden hacer lo que no debés hacer.** No hay parámetro de columna en `mover_tarea`, no hay
  borrado, no hay forma de escribir la descripción de un humano. Si buscás la herramienta para algo
  y no existe, la respuesta suele ser que eso no es tuyo.

**REST**, para lo que las herramientas no exponen: adjuntos, ramas, repos, responsables, usuarios,
carpetas, `/contract` y `/guia`. Y como plan B si las herramientas **no** aparecen — el caso típico
es que el plugin se haya instalado sin token y el servidor no haya conectado. Si te pasa, **decíselo
a tu humano** en vez de arrastrarte por curl toda la sesión: lo arregla en un minuto.

Para el camino REST hace falta además que el token esté en tu **shell** como `$TRELLOUTLOOK_PAT`
(ver punto 2 de abajo). Si las herramientas andan pero los curls te dan 401, es eso y no un problema
de permisos.

El criterio no cambia entre los dos caminos: quién mueve qué, dónde va el avance, qué hacés cuando
te trabás. Eso es el resto de este documento y vale igual. Lo que sí cambia es que **por REST el
servidor te va a frenar con 403** en varias cosas (ver *Errores*), porque las reglas del flujo no
son solo convención: están hechas cumplir en el router.

## Antes de la primera llamada

1. `export PYTHONIOENCODING=utf-8 MSYS_NO_PATHCONV=1` **siempre, primero**. Sin la primera, imprimir el board revienta con `UnicodeEncodeError` (cp932) al llegar a la `Ó` de REVISIÓN. Sin la segunda, en Git Bash `/auth/me` se convierte en `C:/Program Files/Git/auth/me` y urllib tira `InvalidURL`: **entrecomillar el path no evita nada**, solo la variable lo frena. Y ojo, las llamadas con `?` (`/board?board_id=…`) sobreviven igual: por eso el error parece intermitente y terminás descartando el shell como causa justo cuando te falla `/auth/me` o el detalle de una tarjeta. (Efecto lateral: con `MSYS_NO_PATHCONV=1`, `curl -o /dev/null` deja de andar — usá `-o nul`.) **En PowerShell** eso es `$env:PYTHONIOENCODING="utf-8"` — `export` no existe y te tira "The term 'export' is not recognized"; `MSYS_NO_PATHCONV` no hace falta, ahí no hay conversión de rutas.
2. **El token de las herramientas y el de tus curls no son el mismo canal.** El plugin guarda el token en el almacenamiento seguro del sistema y se lo pasa al servidor de herramientas por vos; eso **no** te deja nada en el ambiente. Para llamar la API a mano necesitás `$TRELLOUTLOOK_PAT` exportado en tu shell, y si no está, **pedíselo a tu humano** en vez de buscarlo por ahí. **Nunca** lo escribas en un archivo del repo, en un commit ni en un comentario de tarjeta: los comentarios los ven los clientes del tablero.
3. Base: `https://lectorcorreotrello2-backend.fly.dev/api/v1`, header `Authorization: Bearer <PAT>`. (El `X-Panel-Token` del token de agente legacy todavía responde 200 en instalaciones viejas, pero está retirado: no lo uses, el día que se caiga toda la skill da 401 y la glosa de errores te manda a perseguir un PAT que está perfecto.)
4. **Quién sos y de qué lado estás.** Con herramientas, cualquier respuesta trae `yo` y ahí viene resuelto. Por REST, `GET /auth/me` confirma `agent_token: true` y con qué identidad firmás. El lado (`back` o `front`) lo resuelve el servidor por el nombre de tu token o por el mail de tu humano: **no lo adivines ni lo deduzcas del proyecto**. Si el servidor no lo pudo resolver te lo dice con esas palabras, y ahí lo arregla un humano — no elijas uno por tu cuenta.

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

Los dos recortes de la impresión no son cosméticos. El detalle de una tarjeta ya trabajada puede pesar más de 10 KB: la `description` sola y el historial de `events` te empujan `metadata`, `assignees`, `attachments` y `branches` fuera del corte de 4000, o sea leés media descripción y no ves ni los responsables, ni las capturas, ni las ramas — justo lo que "Trabajá la tarea X" te manda leer. Con el recorte entran `metadata`, `assignees` y `attachments`, y te quedan los últimos 2 eventos, que es lo que necesitás para saber quién la tocó. `branches` es la **última** clave del detalle y en una tarjeta gorda se corta igual: si te importan las ramas, pedí la tarjeta con `OUT=`. Y el `pop` del doc rico **no es hipotético**: buena parte de las tarjetas que escribe un humano desde el panel son `description_format: rich`. Por eso, en una tarjeta con contenido humano **no mandes `description` sola en un PATCH**: eso descarta el doc rico y sus imágenes embebidas. Tu avance va en un comentario.

### Regla dura de contexto

**Nunca dejes que una respuesta de lista llegue cruda al contexto.** Un tablero con historia se pide con `GET /board?board_id=` y puede pesar cientos de KB — decenas de miles de tokens, o más de cien mil en el peor caso —, y `/tasks?board_id=` de ese mismo tablero, casi lo mismo. No hay recorte del lado del servidor: `limit`, `fields` y `column_id` **no existen** y `?q=` **no busca** (responde 200, ignora el texto y devuelve todo). El `[:4000]` del helper es lo que te salva esos tokens por llamada exploratoria; no lo borres. (`/tasks` sí tiene un tope duro de 500 filas del lado del servidor, pero es tan alto que en la práctica no te protege de nada.)

Cuando necesites la lista entera, bajala a archivo y filtrala en python **abriendo con `encoding="utf-8"`**:

```
OUT=t.json python tl.py GET "/tasks?board_id=<id>"
json.load(open("t.json", encoding="utf-8"))     # PYTHONIOENCODING arregla el print, NO el open
```

Sin ese `encoding` explícito el `json.load` muere con `UnicodeDecodeError` cp932 y parece que el archivo se bajó corrupto. El peso depende del tablero: uno recién arrancado es chico y uno con años de historia es el que te come el contexto, así que medí antes de imprimir en vez de suponer.

- `GET /contract` es la fuente de verdad de la API, siempre al día (`X-Contract-Sha`): leelo cuando necesites la forma exacta de un request en vez de suponerla. `GET /guia` es la fuente canónica y versionada del **protocolo de la bandeja** (`X-Guia-Sha`): pedila antes de tocar un dev-prompt, no reconstruyas el protocolo de acá.
- Los dos son **markdown, no JSON**: bajalos a archivo (`OUT=guia.md python tl.py GET "/guia"`) y leé el archivo. `get()` sobre ellos tira `JSONDecodeError`.
- `GET /version` → `{git_sha, contract_sha, guia_sha}`.

## Tableros y columnas

**No hardcodees ids ni nombres de tablero: descubrilos siempre.** El mapa sale de `GET /boards`, que es liviano (unos pocos KB): trae cada tablero visible con su `columns_summary` (`{id, name, kind, tasks_count}` de **todas** sus columnas) y sus `repos`. Pedí `GET /board?board_id=` **solo cuando necesites las tareas**, y a archivo.

```python
from tl import get              # tl.py en el cwd, con TRELLOUTLOOK_PAT exportado
PROYECTO = "…"                  # el nombre que te dijo tu humano
b = next(x for x in get("/boards")["boards"] if x["name"] == PROYECTO)
cols = {c["name"]: c["id"] for c in b["columns_summary"]}
ABIERTO    = next(c["id"] for c in b["columns_summary"] if c["kind"] == "abierto")
EN_PROCESO = cols["EN PROCESO"]
PRUEBA     = cols["PRUEBA INTERNA"]
```

- ABIERTO se resuelve por `kind == "abierto"` (hay exactamente una por tablero): es el único ancla que no depende del nombre. (En `GET /board` el mismo ancla se llama `is_default_inbox`.)
- **Para el resto el `kind` no distingue nada**: `en_curso` son al menos dos (`EN PROCESO` y `PRUEBA INTERNA`), y un tablero puede tener columnas propias con ese mismo kind que son territorio humano. `revision` también suele ser más de una. Mirá el **nombre**.
- **Nunca matchees nombres de columna por substring ni `startswith`**: `REVISIÓN CLIENTE OK` empieza con `REVISI…`. Igualdad exacta. Y los nombres **varían entre tableros**: el mismo estado puede llamarse distinto en dos proyectos, porque un humano puede renombrar una columna. Esta regla es para columnas; para buscar tarjetas por texto, mirá "Trabajá la tarea X".
- Toda columna que no sea ABIERTO / EN PROCESO / PRUEBA INTERNA es **territorio humano** — con una
  excepción: la de rechazo del cliente (`kind: "rechazado"`), de donde sí podés retomar una tarjeta
  si tu humano te lo pide.
- **Las columnas no las tocás**: crearlas, renombrarlas, reordenarlas o cambiarles el `kind` es
  `403`. Son la estructura del tablero, y el techo de abajo se apoya en ellas.

## Quién mueve qué — la regla que no se rompe

| Columna | La mueve |
|---|---|
| ABIERTO | el humano, o **vos** (hallazgo nuevo / devolución a la otra parte) |
| EN PROCESO | **vos**, al arrancar. Verla ahí ya avisa que alguien la está haciendo |
| PRUEBA INTERNA | **vos**, al terminar — pero solo si la tarjeta ENTERA está lista, no solo tu mitad. **Es tu estado terminal: nunca pasás de acá** |
| La de rechazo del cliente (`kind: "rechazado"`) | **vos**, y solo si tu humano te lo pide: es la única salida de la zona del cliente que sigue siendo tuya |
| REVISIÓN y todo lo que sigue (`revision`, `ok`, `cerrado`) | **solo humanos**, en los dos sentidos: ni las llevás ahí ni las sacás de ahí |

REVISIÓN es la puerta al **cliente final**, no una revisión interna: un humano prueba primero. No hay claim para tarjetas comunes (el claim existe solo para los dev-prompts).

**A REVISIÓN solo va una tarjeta 100% terminada**, lista para probarse ENTERA. No "la mitad que
a mí me tocaba": la tarjeta completa, funcionando de punta a punta, como la va a ver el
cliente. Si algo de su alcance no está —típicamente el back está pero la pantalla no—, no es
que esté "casi": **no está**, y su lugar es ABIERTO, para que la trabaje la parte que
corresponda.

**Y esto sí es tuyo, aunque vos no muevas a REVISIÓN.** Es el error más común del tablero y la
forma de cometerlo es esta: dejás en PRUEBA INTERNA una tarjeta a la que le falta la otra
mitad. PRUEBA INTERNA es la **antesala** de REVISIÓN, no un cajón de "listo lo mío": el humano
prueba lo que puede ver, le funciona, y la empuja al cliente. Para cuando se descubre que
faltaba la pantalla, la tarjeta ya cruzó y sacarla de ahí no lo podés hacer vos.

O sea que la pregunta antes de mover a PRUEBA INTERNA no es *"¿terminé?"* sino **"¿si esto se
va al cliente tal como está, se sostiene?"**. Si la respuesta es no, va a ABIERTO con la
devolución completa (ver *"Terminé"*), no a PRUEBA INTERNA.

**Esto no es disciplina, es el servidor.** Con tu PAT, por REST, un movimiento que entre o salga de
esa zona responde **403**, y también crear una tarjeta directamente ahí con `column_id`. Se valida
el **origen y el destino**: sacar una tarjeta de la zona del cliente le borraría de encima el
veredicto que dejó, y por eso duele igual que meterla. La excepción es la columna de rechazo, que es
justamente el camino de vuelta al equipo. Con herramientas ni se plantea: `mover_tarea` no tiene
parámetro de columna, y sus tres destinos son ABIERTO, EN PROCESO y PRUEBA INTERNA.

## Endpoints

```
GET   /boards                  árbol: tableros con columns_summary y repos (liviano)
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

- `metadata` hace **shallow-merge** en PATCH: mandar `{"area":"FRONT"}` conserva `prioridad` y el resto. En uso: `prioridad` (`ALTA|MEDIA|NORMAL`) y `area` (ver *De qué lado estás* más abajo). **Por REST son convención, no enum**: el backend guarda el `metadata` de una tarjeta común tal cual, y un valor mal escrito entra con 200 y deja una tarjeta que el panel no sabe pintar. Copialos exactos. Las herramientas sí los validan y te devuelven la lista de los válidos en el error.
- `move` **no cruza tableros**: con una columna de otro tablero responde un **`404` genérico**, no un 400 explicativo. Ante un 404 en `move`, mirá primero de qué tablero es la columna antes de sospechar de permisos.
- `POST /tasks` con `column_id` tiene **el mismo techo que el move**: con una columna de la zona del cliente responde `403` y no crea nada. Sin `column_id` nace en el ABIERTO del tablero, que es lo que querés casi siempre.
- La lista plana es liviana: `assignees` y `comments_count` vienen vacíos, y los `attachments` (capturas que dejó el humano) solo aparecen en el detalle. Al revés, el detalle trae `column_id` pero **no** `column_name`. `?assignee=<uuid>` sí filtra.

## De qué lado estás

El flujo reparte trabajo entre **dos partes**, y hoy el backend las nombra con dos literales fijos:
`back` y `front`. No son "una persona" ni "un proyecto": son los dos lados entre los que se pasan
pedidos, y el servidor **rechaza cualquier otro valor con 400**. Tres consecuencias prácticas:

- **Tu lado no lo elegís vos**: lo resuelve el servidor (ver *Antes de la primera llamada*, punto 4).
- Cuando **vos dejás** un pedido, `prompt_para` es el **otro** lado. Cuando **buscás tu turno**,
  filtrás por el **tuyo**. Con `buscar_tareas bandeja="mi_turno"` eso ya viene resuelto.
- `metadata.area` es un campo **distinto** y no es lo mismo: etiqueta a qué parte le toca una
  **tarjeta** (no un pedido). Las herramientas aceptan hoy `BACK`, `FRONT` y `BACK-FRONT`; si
  mandás otra cosa, el error te lista las válidas.

Si el equipo suma una tercera parte (administración, datos, diseño), esto **no** alcanza: hace falta
tocar el backend. Mientras tanto, no inventes valores nuevos ni los escondas en el texto libre.

---

# Situaciones

## "Trabajá la tarea X"

**Con herramientas:** `buscar_tareas` con `texto=` para ubicarla, `ver_tarea` para leerla entera,
`mover_tarea accion="empezar"` para arrancar (y `retomar=true` si venía rechazada por el cliente).
Tres llamadas y ninguna te cuesta el tablero. Lo de abajo es la variante por REST.

Si te la nombran por título y no por id: `OUT=t.json python tl.py GET "/tasks?board_id=<id>"` y filtrá por **substring en minúsculas sobre `title` y `description`** — la lista plana trae los dos y muchas veces el tema está solo en la descripción. (La igualdad exacta es la regla de los nombres de columna, no la de buscar tarjetas.)

`GET /tasks/{id}` (leé `description`, `metadata`, `assignees`, `attachments`, `branches`) + `GET /tasks/{id}/comments` para el hilo. Como el detalle no trae el nombre de la columna, resolvé su `column_id` contra el mapa que ya armaste **antes de mover**:

- **En ABIERTO** → `POST /tasks/{id}/move` a `EN_PROCESO` antes de tocar código.
- **Ya en EN PROCESO** → no la muevas: el move a la misma columna sale 200, la sube al tope y les avisa a los humanos un arranque falso. Alguien la está haciendo: leé el último comentario y `events`, y confirmá con tu humano que la seguís vos.
- **En la columna de rechazo del cliente** (`kind: "rechazado"`) → esa sí la podés retomar a EN PROCESO, y **solo** si tu humano te lo pide: el cliente la devolvió y el trabajo vuelve al equipo.
- **En REVISIÓN o más allá** → no la toques. Por REST el servidor te contesta **403** (se valida el origen, no solo el destino) y la herramienta directamente no tiene por dónde. La saca un humano desde el panel: pedíselo y decile por qué.

No toques `assignees` al arrancar: el move a EN PROCESO ya es la señal, y el PUT reemplaza la lista entera (podés borrar a un responsable real que no es tu humano).

## "Terminé"

**Con herramientas:** `mover_tarea accion="terminar"` con la nota — mueve y comenta en un solo paso.

Comentario con qué hiciste, qué archivos tocaste, **cómo verificarlo**, **dónde quedó** (rama / PR; si deployaste, el `git_sha` de `GET /version`) y qué quedó afuera → `move` a `PRUEBA INTERNA` → **ahí frenás** y le avisás al humano que quedó para que pruebe.

Si el proyecto tiene repos cargados (vienen en `repos` de `GET /boards`), registrá la rama: `POST /tasks/{id}/branches {"repo_id":"…","branch":"…"}`, con un repo de **ese** tablero. Una rama por repo y tarea: la segunda da 409 y se edita con `PATCH /tasks/{id}/branches/{branchID}`. Si el proyecto no tiene repos, la rama va en el comentario.

Antes de mover, preguntate si está lista la tarjeta **entera**, y no solo si `metadata.area` te nombra a vos. El área es una etiqueta y se queda vieja: puede decir BACK mientras el «qué hay que hacer» de la descripción pide una pantalla que nadie hizo. **Lo que manda es el alcance escrito en la tarjeta**, y muchas veces esa misma descripción ya dice quién sigue («coordinar con el front», «prompt para=front al cerrar el back»): eso es una instrucción, no un comentario al pasar. Si de ese alcance queda algo para la otra parte, PRUEBA INTERNA es mentira. Comentás lo tuyo y hacés la devolución completa: `move` a **ABIERTO** (la tarjeta está en EN PROCESO, la moviste vos al arrancar — no se queda sola) + `PATCH {"metadata":{"area":"…"}}` con el área que queda trabajando + dev-prompt con `metadata.prompt_tarea_id` = el id de esa tarjeta; es la receta de "Me trabé: depende de la otra parte". PRUEBA INTERNA es cuando la tarjeta **entera** está lista para que la prueben. Dicho al revés, que es como se comete el error: dejar ahí una tarjeta a la que le falta la otra mitad no la deja "casi lista", la manda al cliente a medio hacer, porque el humano prueba lo que ve, le funciona y la empuja a REVISIÓN. Desde ahí no la podés sacar.

**Y ojo con una trampa que arruina la regla de arriba: una tarjeta AUTOCREADA por un dev-prompt
no describe la función, describe el PEDIDO.** Cuando el backend crea la tarjeta de trabajo de
un prompt, le copia el texto del prompt. Así que su descripción es *la mitad que la otra parte
te pidió a vos*, no lo que la tarjeta tiene que entregar — y si buscás ahí «el alcance
escrito», encontrás una lista de endpoints y concluís que terminaste.
La pista está igual, y casi siempre en las primeras líneas, disfrazada de contexto: «el
rediseño está entregado completo **salvo la sección X**», «la UI ya está hecha **y apagada**».
Eso no es motivación: **es el alcance**. La tarjeta existe para que exista X, y tus endpoints
son un insumo.
Cómo se detecta sin pensarlo: la pregunta no es *«¿entregué lo que el prompt pedía?»* sino
**«¿alguien puede USAR esto ahora?»**. Si la respuesta es que falta la pantalla, falta la
tarjeta.
**Responder el prompt y mover la tarjeta son dos decisiones distintas, y se toman por
separado.** Responder `cierra` dice «tu pedido está entregado» y es cierto. Mover a PRUEBA
INTERNA dice «esto se puede probar entero», que es otra cosa. Coinciden solo cuando el alcance
de la tarjeta es exactamente lo que el prompt pedía — y en una tarjeta autocreada casi nunca lo
es. Cerrá el prompt igual, y a la tarjeta devolvela con la receta de siempre.

El avance va **en un comentario**, no reescribiendo la descripción: el pedido original del humano queda intacto. `description` **no** hace shallow-merge, un PATCH la reemplaza entera. Solo la tocás si está **vacía**: `PATCH {"description":"..."}` en texto plano (no armes `description_rich`). Si está **a medias pero con contenido del humano, no la patchees**: mandar solo `description` descarta el doc rico y libera sus imágenes embebidas, o sea le borrás el formato y las capturas. Tu complemento va en un comentario.

## "Me trabé: necesito algo del humano"

Comentás exactamente qué te frena y qué necesitás para seguir. **Dejás la tarjeta en EN PROCESO** (sigue siendo tuya).

## "Me trabé: depende de la otra parte"

**Con herramientas:** `mover_tarea accion="devolver"` con `nota=` y `pedido=` hace los cuatro pasos
de una vez y sin que puedas olvidarte del enlace. Si es una vuelta nueva de un hilo que ya existe,
pasá `responde_a=<id del prompt anterior>`: encadena y cierra el eslabón viejo.

**Paso cero, antes de escribir el pedido: leé el hilo que ya existe.**
`GET /tasks/{id}/prompts`, con el id de la **tarjeta**. Una tarjeta con historia ya fue y volvió, y
el prompt cerrado de la vuelta anterior te dice dos cosas que la descripción no dice: **qué
entregó ya la otra parte** —volver a pedirlo la manda a rehacer trabajo hecho, y encima firmado
por tu humano— y **qué quedó parado a propósito**, que casi siempre es una condición que
acordaron entre ustedes («esto cuando exista la segunda org») y que puede seguir sin cumplirse.
Si lo que reactivás estaba parado por acuerdo, **decilo en el pedido** y dejale la salida de
responder `requiere` para sostener el orden pactado: reactivar no es exigir.

Y si es una vuelta nueva de ese hilo, el prompt nuevo lleva `metadata.prompt_padre` = el id del
anterior. `prompt_padre` no es solo para cuando no estás de acuerdo: es para toda vuelta que
continúa un prompt ya cerrado.

Por REST son cuatro pasos y hay que hacerlos todos: comentás qué falta → `move` a **ABIERTO** con las instrucciones → `PATCH {"metadata":{"area":"…"}}` → dev-prompt para el agente de esa parte **con `metadata.prompt_tarea_id` = el id de esa misma tarjeta**. Si lo omitís, el backend crea una tarjeta de trabajo nueva: te quedan dos para lo mismo y el hilo partido.

## "Encontré un bug o deuda al pasar"

**Con herramientas:** `crear_tarea` (nace en ABIERTO del proyecto que elijas; con `pedido=` deja
además el dev-prompt enlazado). No acepta columna a propósito: una tarjeta nueva nace en ABIERTO.

No lo arregles de prepo. Tarjeta nueva en ABIERTO del **mismo proyecto** (qué pasa, dónde, cómo reproducirlo, que salió trabajando la tarjeta X) y seguís con lo tuyo. **Mandá siempre `board_id`**: sin él la tarea cae en el tablero principal, o sea en el proyecto equivocado.

## "¿Qué hay pendiente?"

**Con herramientas:** `buscar_tareas` sin filtros ya te devuelve el trabajo activo de todos los
proyectos, compacto y sin los dev-prompts mezclados. Es la respuesta entera a esta pregunta.

Por REST: `GET /tasks?board_id=<a>,<b>` (varios ids separados por coma en **una sola llamada**; cada tarea trae `board_name`) → filtrás `column_name` = ABIERTO **y tirás las que tengan `metadata.tipo == "dev-prompt"`**. El kanban las oculta a propósito (no entran en `columns_summary`): son coordinación entre agentes, no trabajo pendiente. Sin ese descarte le reportás como pendiente la coordinación entre agentes —que en un tablero activo puede ser la mayoría del ABIERTO— y encima algún prompt repite el título de su tarjeta enlazada: tu resumen no coincide con lo que él ve en pantalla. Resumís y podés sugerir por dónde empezar; **no arranques**.

## Bandeja de dev-prompts (entre las dos partes)

Un dev-prompt es una **tarea** con `metadata.tipo: "dev-prompt"`. **Viven todos en el tablero principal** y el proyecto del que hablan va en `metadata.prompt_proyecto`, que es lo que dice `/guia` y lo que hace el panel. Está decidido que en el futuro cada uno viva en el tablero de su proyecto, pero eso espera a que exista mover tareas entre tableros y a que `/guia` lo diga: **hasta entonces, creálos en el principal**, o el otro agente no los va a encontrar.

**Todo lo de la bandeja va adentro de `metadata`** (que hace shallow-merge): `prompt_para`, `prompt_estado`, `prompt_autor`, `prompt_proyecto`, `prompt_tarea_id`, `prompt_respuesta`, `prompt_respuesta_tipo`, `prompt_prioridad`, `prompt_padre`. Ninguno existe en la raíz de la tarea. El PATCH con los campos sueltos responde **200 sin cambiar nada** (el handler solo lee `title`, `description`, `metadata`), así que creés que respondiste y el prompt sigue pendiente con el claim tomado. Y el POST **solo te frena con 400 si `metadata` trae `tipo: "dev-prompt"`**: si mandás todo suelto y sin `metadata`, responde **201** y te deja una tarjeta común que nadie va a leer como prompt.

**Con herramientas, la bandeja entera son dos:** `buscar_tareas bandeja="mi_turno"` para ver qué te
toca (`bandeja="todos"` para el panorama) y `atender_prompt` con `accion` `tomar` / `responder` /
`cerrar` para el ciclo. Esa herramienta tiene adentro la tabla de verdad completa del protocolo
—quién puede hacer qué según el estado y el lado—, así que si te frena, la razón que te da es la
correcta y no hace falta que la deduzcas. Lo de abajo es la vía REST, y es la que tenés que conocer
igual para todo lo que la herramienta no cubre.

La bandeja **también la dispara el humano**: podés listar los prompts de tu lado y avisar qué hay, pero correrlos te los pide él. El protocolo completo —campos, enums, estados, hilos— está en `GET /guia`.

- **Ver tu turno:** `OUT=p.json python tl.py GET "/tasks?board_id=<ids separados por coma>"` (una sola llamada, con `board_name`) y sobre los `metadata.tipo=="dev-prompt"` filtrás dos cosas: **(a)** `metadata.prompt_estado=="pendiente"` con `metadata.prompt_para` = **tu** lado, que son los que te piden; **(b)** los que **pediste vos** (`prompt_para` = el otro lado) que volvieron `"corrido"` con `metadata.prompt_respuesta_tipo == "requiere"`: leés `prompt_respuesta`, hacés lo que pida y cerrás con `PATCH {"metadata":{"prompt_estado":"cerrado"}}`. Los `"cierra"` se cierran solos, esos no son tu turno. Pedir todos los tableros de una es una precaución barata: hoy los prompts viven en el principal, así que da lo mismo, y el día que se repartan por proyecto la receta sigue andando sin tocarla.
- **Dejar uno:** `POST /tasks` con **`board_id` del tablero principal** (ahí viven todos hoy) y `metadata` = `{"tipo":"dev-prompt","prompt_para":"<el otro lado>","prompt_estado":"pendiente","prompt_proyecto":"…"}`, más una descripción con la forma exacta de lo que necesitás (endpoint, campos, ejemplo) y qué ya está hecho de tu lado. `prompt_proyecto` **etiqueta, no ubica**: el ejemplo de `/guia` omite `board_id` y sin él el prompt y su tarjeta caen en el tablero principal. Sin `column_id` nace en el ABIERTO de ese tablero.
- **La tarjeta que el backend autocrea lleva el texto del prompt**, así que describe el PEDIDO y no la función. Cuando te toque terminarla, no leas ahí «el alcance»: leé qué dice que falta (ver *"Terminé"*).
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

- `400` body inválido o valor fuera de enum — el mensaje dice qué arreglar, leelo. Ojo: por REST los únicos enums que el backend valida son los `prompt_*` de la bandeja; el `metadata` de una tarjeta común se guarda tal cual. Esos `prompt_*` son **insensibles a mayúsculas y espacios** (`" BACK "` entra y queda guardado `back`), pero un valor que no esté en el enum sigue siendo 400.
- `401` falta o no sirve el token — **excepto `/auth/tokens`, que responde 401 aunque tu PAT esté perfecto**: acuñar o listar tokens exige la sesión del humano en el panel. El PAT tampoco sirve para el WebSocket (401 por diseño): todo por REST, y para ver cambios volvés a pedir.
- `403` **algo que un token de agente no hace**. No es un permiso que se pueda pedir: son operaciones
  que exigen un humano en sesión, y el mensaje siempre dice cuál es y qué hacer. Cuatro familias:
  - **Gestión**: miembros, visibilidad, dueños, roles, organizaciones, carpetas privadas, tokens.
  - **Techo de columna**: mover o crear una tarjeta fuera del trabajo interno (ABIERTO / EN PROCESO /
    PRUEBA INTERNA), en las dos direcciones. Excepción: retomar una rechazada por el cliente.
  - **Borrar**: tableros, carpetas, columnas, tarjetas, comentarios, adjuntos, repos. (Las ramas de
    una tarea sí las podés borrar: son tuyas y se rehacen con un POST.)
  - **Columnas**: crearlas, renombrarlas, reordenarlas o cambiarles el `kind`.

  Si un `403` te frena en algo que tu humano te pidió, no busques la vuelta: pasale el texto del
  error, que está escrito para que se entienda de una, y lo hace él desde el panel.
- `404` **"no existe O no lo ves"** — deliberado, no es un bug. Preguntate si ese tablero o tarea es privado para tu humano; y si fue un `move`, si la columna era de otro tablero.
- `409` conflicto de estado (claim ya tomado, segunda rama del mismo repo).

## Lo que no hacés nunca

Estas cuatro **no las hacés aunque el servidor te deje**. Son criterio, y por eso están acá:

- Arrancar una tarea que el humano no te pidió.
- Pisar la descripción original de un humano.
- Escribir el PAT en cualquier lado (un archivo del repo, un commit, un comentario de tarjeta).
- Tocar prompts o tarjetas de un proyecto que no es el tuyo.

Y estas el servidor directamente no te las deja (`403`), así que si te tienta alguna, la salida es
pedírsela a tu humano y no buscarle la vuelta: cruzar a REVISIÓN o más allá (o sacar algo de ahí),
borrar cualquier cosa, y tocar las columnas de un tablero.
