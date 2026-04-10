# Asistente RAG para Foros de Moodle UTN

## Resumen del Proyecto

Sistema de automatización basado en **n8n** que monitorea los foros de Moodle de la UTN (TUPaD — Tecnicatura Universitaria en Programación a Distancia), detecta preguntas sin responder de estudiantes, genera respuestas sugeridas usando **RAG** (Retrieval-Augmented Generation) con los PDFs del curso, y permite al tutor aprobar o rechazar las respuestas vía **Telegram** antes de publicarlas en el foro.

---

## Contexto

- **Institución**: UTN — Tecnicatura Universitaria en Programación a Distancia (TUPaD)
- **Rol del usuario**: Tutor de la cátedra
- **Plataforma Moodle**: [https://tup.sied.utn.edu.ar](https://tup.sied.utn.edu.ar)
- **Course ID**: 38
- **Instancia n8n**: [https://belmontelucero-n8n.326kz3.easypanel.host](https://belmontelucero-n8n.326kz3.easypanel.host) (desplegada en Easypanel)

---

## Arquitectura General

El sistema se compone de **3 workflows** en n8n:

```
┌─────────────────────┐
│  Workflow 1          │
│  INGESTA RAG         │
│  (Manual/On-demand)  │
│                      │
│  Google Drive → PDF  │
│  → Extract → Filter  │
│  → Embed → Pinecone  │
└─────────────────────┘

┌─────────────────────┐
│  Workflow 2          │
│  MONITOR + RAG       │
│  (Cada 30 min)       │
│                      │
│  Moodle API → Foros  │
│  → Sin respuesta?    │
│  → RAG query         │
│  → LLM genera        │
│  → Telegram + botones│
└─────────────────────┘

┌─────────────────────┐
│  Workflow 3          │
│  REPLY               │
│  (Telegram Trigger)  │
│                      │
│  Callback "Enviar"   │
│  → HTTP scraping     │
│  → Post en Moodle    │
│  → Confirmación      │
└─────────────────────┘
```

---

## Stack Tecnológico

| Componente | Tecnología | Detalles |
|---|---|---|
| Orquestador | n8n (self-hosted) | Easypanel |
| Vector Store | Pinecone (free tier) | Index: `moodle-utn`, dims=1024, metric=cosine |
| Embeddings | Mistral Embed | `mistral-embed`, 1024 dimensiones |
| LLM | Gemini Flash | Para generación de respuestas |
| Fuente de PDFs | Google Drive | Una carpeta por unidad |
| Notificaciones | Telegram Bot | Inline keyboard para aprobar/rechazar |
| Plataforma educativa | Moodle (mobile web services) | REST protocol deshabilitado, mobile habilitado |

---

## Workflow 1 — Ingesta RAG

**Trigger**: Manual
**Propósito**: Ingestar PDFs del curso en Pinecone para alimentar el sistema RAG.

### Flujo de Nodos

```
Manual Trigger → Configure → List Folders (Drive) → List PDFs → Download
    → Extract PDF Text → Filter (>100 chars) → Embed (Mistral) → Pinecone Upsert
```

### Arquitectura Defensiva

La pipeline separa **extracción, validación y embedding** en pasos distintos para que las fallas en una etapa no contaminen las siguientes:

1. **Extract from File** (operación PDF): extrae texto crudo del PDF
2. **Filter** (umbral: `text.length > 100`): descarta PDFs vacíos o con extracción fallida
3. **Default Data Loader** (modo JSON): recibe el texto pre-extraído vía expresión `{{ $json.text }}`
4. **Mistral Embeddings** (1024 dims): genera vectores del texto extraído
5. **Pinecone Upsert**: almacena vectores con metadata `{unidad, filename, chunk_index}`

**Razón**: Si el loader procesa binario directamente y falla la extracción, genera chunks vacíos que producen vectores de dimensión 0 al llegar a Pinecone. Separando los pasos, cada etapa es observable en el debug de n8n.

### Metadata en Pinecone

Cada vector almacenado incluye:

| Campo | Descripción | Ejemplo |
|---|---|---|
| `unidad` | Número de unidad (1-10) | `3` |
| `filename` | Nombre del PDF de origen | `repetitivas_teoria.pdf` |
| `chunk_index` | Índice del chunk dentro del PDF | `0`, `1`, `2`... |

Esta metadata permite filtrar por unidad al momento de la query RAG, evitando ruido entre temas.

### PDFs por Unidad

| Unidad | Tema | Cantidad de PDFs |
|---|---|---|
| 1 | Estructuras Secuenciales | 6 |
| 2 | Estructuras Condicionales | 4 |
| 3 | Estructuras Repetitivas | 4 |
| 4 | Git / Trabajo Colaborativo | 4 |
| 5 | Listas | 7 |
| 6-10 | (Pendiente de monitoreo) | — |

**Total unidades 1-5**: 25 PDFs

---

## Workflow 2 — Monitor + RAG

**Trigger**: Schedule (cada 30 minutos)
**Propósito**: Detectar preguntas sin responder en los foros monitoreados, generar respuesta sugerida vía RAG, enviar al tutor por Telegram con botones de aprobación.

### Flujo de Nodos

```
Schedule (30 min)
    → Obtener Token (Moodle mobile API)
    → Get Forums (mod_forum_get_forums_by_courses)
    → Filter Target Forums (solo CMIDs monitoreados)
    → Get Discussions (por foro, mod_forum_get_forum_discussions)
    → Find Unanswered (Code node: filtra numreplies === 0)
    → Embed Pregunta (Mistral Embed, 1024 dims)
    → Pinecone Query (top-k, filtrado por metadata.unidad)
    → Gemini Flash (prompt con contexto RAG + pregunta del alumno)
    → Telegram (mensaje con respuesta sugerida + inline keyboard)
```

### Detalle del Flujo RAG

1. **Embedding de la pregunta**: la pregunta del alumno se embede con Mistral Embed (mismo modelo que la ingesta) para garantizar compatibilidad dimensional
2. **Query a Pinecone**: búsqueda por similitud con filtro `unidad = N` (donde N corresponde al foro de origen). Se recuperan los top-k chunks más relevantes
3. **Prompt al LLM**: se construye un prompt con:
   - Contexto: chunks recuperados de Pinecone
   - Pregunta original del alumno
   - Instrucciones: responder como tutor, tono educativo, en español
4. **Telegram**: el mensaje incluye la pregunta del alumno, la respuesta sugerida, y botones inline:
   - **"Enviar"** → `callback_data: enviar:DISCUSSION_ID`
   - **"Ignorar"** → `callback_data: ignorar:DISCUSSION_ID`

### Acceso a la API de Moodle

- **Autenticación**: POST a `/login/token.php` con `service=moodle_mobile_app`
- **Funciones disponibles**:
  - `mod_forum_get_forums_by_courses` — listar foros del curso
  - `mod_forum_get_forum_discussions` — obtener discusiones de un foro
- **Función NO disponible**: `mod_forum_add_discussion_post` — por eso se usa HTTP scraping para publicar respuestas

### Foros Monitoreados

| CMID | Forum ID | Tema | Unidad |
|---|---|---|---|
| 11156 | 1357 | Estructuras Secuenciales | 1 |
| 11201 | 1359 | Estructuras Condicionales | 2 |
| 11226 | 1360 | Estructuras Repetitivas | 3 |
| 11181 | 1358 | Git / Trabajo Colaborativo | 4 |
| 11248 | 1361 | Listas | 5 |

> **Nota**: Los CMIDs en las URLs de Moodle (`view.php?id=X`) son IDs de módulo de curso, NO IDs internos de foro. Se usa `mod_forum_get_forums_by_courses` para mapear CMID → ID.

---

## Workflow 3 — Reply (Publicar Respuesta)

**Trigger**: Telegram Trigger (callback_query)
**Propósito**: Cuando el tutor presiona "Enviar" en Telegram, publicar la respuesta generada en el foro de Moodle y confirmar la acción.

### Flujo de Nodos

```
Telegram Trigger (callback_query)
    → Parse Callback Data (Code node: extraer acción + discussion_id)
    → IF acción === "enviar"
        ├─ YES:
        │   → Extract Respuesta (del texto del mensaje de Telegram)
        │   → Login Moodle (HTTP Request: POST /login/index.php)
        │   → Extract Cookies (Set-Cookie headers)
        │   → Get Discussion Page (HTTP Request: GET /mod/forum/discuss.php?d=DISCUSSION_ID)
        │   → Extract Sesskey (regex sobre HTML: sesskey=XXXXX)
        │   → Post Reply (HTTP Request: POST /mod/forum/post.php)
        │       Body: sesskey, message, discussion, reply (parent post ID)
        │   → Edit Telegram Message ("Respuesta enviada al foro")
        │
        └─ NO (ignorar):
            → Edit Telegram Message ("Consulta ignorada")
```

### Detalle del Flujo de Publicación

El flujo de publicación en Moodle usa **HTTP scraping** porque la función de API `mod_forum_add_discussion_post` no está habilitada en la instancia.

#### Paso 1 — Login con Cookies

```
POST https://tup.sied.utn.edu.ar/login/index.php
Content-Type: application/x-www-form-urlencoded

username=44662828&password=%25Matyalts135&anchor=
```

Se extraen los headers `Set-Cookie` de la respuesta para mantener la sesión autenticada.

#### Paso 2 — Obtener Sesskey

```
GET https://tup.sied.utn.edu.ar/mod/forum/discuss.php?d={DISCUSSION_ID}
Cookie: {cookies del paso anterior}
```

Del HTML de respuesta se extrae el `sesskey` con regex: `"sesskey":"([a-zA-Z0-9]+)"` o del input hidden.

#### Paso 3 — Publicar Respuesta

```
POST https://tup.sied.utn.edu.ar/mod/forum/post.php
Cookie: {cookies}
Content-Type: application/x-www-form-urlencoded

sesskey={sesskey}&reply={parent_post_id}&discussion={discussion_id}&message[text]={respuesta}&message[format]=1
```

#### Paso 4 — Confirmación

Se edita el mensaje original de Telegram para reflejar el resultado:
- Exitoso: "Respuesta publicada en el foro"
- Error: "Error al publicar — revisar manualmente" + link al foro

### Notas de Implementación

- **Cookies entre requests**: hay que extraer manualmente los headers `Set-Cookie` y pasarlos como header `Cookie` en cada request siguiente. n8n no maneja sesiones automáticamente.
- **Telegram callback_data** tiene límite de 64 bytes: la respuesta completa se recupera del texto del mensaje original de Telegram (no cabe en el callback).
- **Sesskey** es un token CSRF de Moodle, cambia por sesión. Se debe obtener fresh en cada flujo de publicación.
- El `parent_post_id` se necesita para responder al post correcto dentro de la discusión. Se puede obtener del HTML de la página de discusión.

---

## Decisiones Técnicas y Gotchas

### Elección de Mistral Embed sobre Gemini

Se eligió **Mistral Embed** (`mistral-embed`, 1024 dims) sobre Gemini por las siguientes razones:

| Aspecto | Gemini embedding-001 | Gemini text-embedding-004 | Mistral Embed |
|---|---|---|---|
| Dimensiones | 3072 | 768 | 1024 |
| RPM (free) | 100 | 1,500 | Generoso |
| TPM (free) | 30,000 | 15,000,000 | Generoso |
| Problema en n8n | Ignora config de modelo | — | Configurable |

**Problemas encontrados con Gemini en n8n**:

1. **El nodo ignora la configuración de modelo**: `embeddingsGoogleGemini` usa `gemini-embedding-001` (3072 dims) por defecto, ignorando `text-embedding-004` configurado en código
2. **Rate limiting silencioso**: `gemini-embedding-001` tiene límites free tier muy restrictivos (30k TPM). Cuando se exceden, Gemini devuelve respuestas vacías/degeneradas en lugar de un error explícito, causando vectores de dimensión 0 en Pinecone
3. **Error misleading**: `"Vector dimension 0 does not match dimension 3072"` aparenta ser un problema de datos vacíos, pero la causa real es rate limiting del API de embeddings

**Lección**: Siempre verificar el dashboard del proveedor de embeddings ANTES de asumir problemas de datos.

### n8n Code Node — Sin fetch()

El Code Node v2 de n8n **no tiene `fetch` disponible** en el sandbox. Todas las llamadas HTTP deben hacerse con nodos HTTP Request, no dentro de Code nodes.

### Manejo de Cookies para Scraping

Para flujos que requieren scraping con cookies (como publicar en Moodle), hay que extraer manualmente los headers `Set-Cookie` de la respuesta y pasarlos como header `Cookie` en los requests siguientes. n8n no tiene manejo automático de sesiones HTTP.

### Forum CMIDs vs Forum IDs

Los CMIDs en las URLs de Moodle (`view.php?id=X`) son IDs de módulo de curso, NO IDs internos de foro. Siempre usar `mod_forum_get_forums_by_courses` para obtener el mapeo correcto.

---

## Estado Actual del Proyecto

### Completado

- [x] Workflow 2 (Monitor) — Detecta preguntas sin responder, notifica por Telegram
- [x] Acceso a API de Moodle vía mobile web services
- [x] Bot de Telegram configurado y funcionando
- [x] Google Drive configurado como fuente de PDFs (una carpeta por unidad)

### En Progreso

- [ ] Workflow 1 (Ingesta RAG) — Migrar de Gemini a Mistral Embed (1024 dims)
- [ ] Índice de Pinecone `moodle-utn` — Recrear con 1024 dimensiones para Mistral

### Pendiente

- [ ] Configurar credencial de Mistral en n8n
- [ ] Completar ingesta de PDFs en Pinecone con Mistral Embed
- [ ] Integración de RAG query en Workflow 2 (embed pregunta + query Pinecone + Gemini LLM)
- [ ] Inline keyboard en Telegram ("Enviar" / "Ignorar") en Workflow 2
- [ ] Workflow 3 (Reply) — Login + sesskey + POST para publicar en Moodle
- [ ] Confirmación en Telegram post-publicación
- [ ] Monitoreo de foros de unidades 6-10

---

## Credenciales Necesarias

| Servicio | Estado | Notas |
|---|---|---|
| Telegram Bot | Configurado | Chat ID: `1415649706` |
| Moodle (mobile API) | Configurado | Usuario/password en nodo "Obtener Token" |
| Mistral API | Pendiente | Para embeddings `mistral-embed` (1024 dims) |
| Google Gemini API | Configurado | Para LLM (generación de respuestas) |
| Pinecone | Pendiente | Recrear index `moodle-utn` con dims=1024 |
| Google Drive | Configurado | Una carpeta por unidad |

---

## Infraestructura

- **n8n**: Self-hosted en Easypanel
- **URL**: https://belmontelucero-n8n.326kz3.easypanel.host
- **Repositorio local**: `G:\Proyectos\n8n-moodle-forums`

---

## Diagrama de Flujo Completo

```
                    ┌──────────────┐
                    │ Google Drive  │
                    │ (PDFs/unidad) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  WF1: Ingesta │
                    │  Extract+Embed│
                    │ (Mistral 1024)│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Pinecone    │
                    │  moodle-utn   │
                    │  (1024 dims)  │
                    └──────┬───────┘
                           │
┌──────────────┐    ┌──────▼───────┐    ┌──────────────┐
│   Moodle     │───▶│  WF2: Monitor │───▶│   Telegram   │
│   Forums     │    │  + RAG Query  │    │  (Botones)   │
│  (cada 30m)  │    │ Mistral+Gemini│    └──────┬───────┘
└──────┬───────┘    └──────────────┘           │
       │                                        │
       │         ┌──────────────┐               │
       │◀────────│  WF3: Reply  │◀──────────────┘
       │         │              │       (callback: Enviar)
       │         │ 1. Login     │
       │         │ 2. Sesskey   │
       │         │ 3. POST reply│
  (respuesta     │ 4. Confirmar │
   publicada)    └──────────────┘
```

---

*Documentación generada el 2026-04-10 — Proyecto n8n-moodle-forums*
