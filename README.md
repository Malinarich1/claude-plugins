# claude-plugins

Plugins de Claude Code del equipo. Hoy hay uno: **trelloutlook**, que le enseña a un agente a
trabajar con nuestro kanban por su API REST.

## Instalar (una vez por máquina)

```
/plugin marketplace add Malinarich1/claude-plugins
/plugin install trelloutlook@malinarich-plugins
```

Hace falta acceso de **lectura** a este repo y que tu `git` ya autentique contra GitHub (SSH con
la clave cargada en el agente, o `gh auth login`). El atajo `owner/repo` clona por SSH.

## Configurar tu token (una vez, y no va en este repo)

Cada uno usa **su** token personal de TrellOutlook, así lo que hace su IA queda atribuido a él.
Se define en tu `settings.json` personal — en Windows, `C:\Users\<vos>\.claude\settings.json`:

```json
{
  "env": {
    "TRELLOUTLOOK_PAT": "lct_pat_..."
  }
}
```

El token se emite desde el panel (sección "Mi IA"). Si no tenés uno, pedíselo a Nicolás. **Nunca**
lo pongas en un archivo de este repo ni en un comentario de una tarjeta.

## Usarlo

La skill se invoca sola cuando le pedís algo del tablero ("trabajá la tarjeta X", "qué hay
pendiente", "dejale un prompt al front"). También podés llamarla a mano con `/trelloutlook:kanban`.

## Actualizar

```
/plugin marketplace update malinarich-plugins
```

No usamos número de versión: cada commit de este repo es la versión. Al actualizar, se toma el
último commit de `main`.

## Qué trae

**La skill** (`/trelloutlook:kanban`): el criterio de trabajo. Qué columna mueve el agente y cuál
no, dónde escribe el avance, qué hace cuando se traba.

**Las herramientas** (servidor MCP): seis verbos del flujo —buscar, ver, mover, comentar, crear y
atender un pedido de la bandeja— servidos por el backend. No hay nada que instalar: se conectan
solas con tu token. Las reglas están dentro de las herramientas, no solo escritas: no existe forma
de mover una tarjeta a REVISIÓN, ni de borrar, ni de reescribir la descripción de un humano.

```
.claude-plugin/marketplace.json          el catálogo (lo que instala el comando de arriba)
plugins/trelloutlook/
  .claude-plugin/plugin.json             el manifiesto del plugin
  .mcp.json                              conexión al servidor de herramientas
  skills/kanban/SKILL.md                 la skill: el criterio de trabajo
```

La skill **no copia el contrato de la API**: para el detalle exacto de un endpoint apunta a
`GET /contract`, que sale del backend y siempre está al día. Lo que vive acá es el criterio que
ninguna herramienta transmite sola: qué columna mueve el agente y cuál no, dónde escribe el
avance, qué hace cuando se traba.

El servidor MCP (herramientas tipadas para cualquier cliente, no solo Claude) va a vivir **dentro
del backend**, y este repo solo va a llevar la configuración que lo apunta. Está como tarjeta.
