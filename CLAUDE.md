# CLAUDE.md — MM Talent Sport · Scouting

Documentación completa del proyecto para Claude Code. Todo el código reside en un único archivo: `index.html`.

---

## Visión general

**MM Talent Sport** es una plataforma web de scouting y analytics de fútbol base española. Permite a scouts gestionar jugadores, registrar visitas a partidos, generar informes profesionales con IA, comparar jugadores, visualizarlos en mapa y mucho más.

- **URL de producción:** `https://mm-talent-sport.vercel.app`
- **Archivo único:** `index.html` (~2980 líneas — HTML + CSS inline + JS module)
- **Idioma:** Español (interfaz y datos)
- **Fundador:** Mario Martínez (`mariop123478@gmail.com`)

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML5 + CSS3 vanilla (inline) + JavaScript ES Module |
| Auth & DB | Firebase 10.12.0 (Auth + Firestore) |
| IA | Anthropic Claude API (`claude-sonnet-4-20250514`) |
| Mapa | Leaflet.js 1.9.4 + OpenStreetMap tiles |
| Gráficos | Chart.js 4.4.4 |
| Radar chart | SVG generado manualmente (no depende de librerías) |
| Tipografías | Google Fonts: Inter (400-800) + Syne (700-800) |
| PWA | Service Worker inline + Web App Manifest dinámico |
| Hosting | Vercel (deploy desde GitHub) |

---

## Arquitectura

### Patrón: Single Page Application (SPA) de un solo archivo

No hay bundler, no hay framework, no hay archivos separados. Todo es `index.html`.

### Sistema de páginas

Las páginas son `<div id="page-X">` con clase `.page`. La función `showPage(id)` activa una quitando/añadiendo la clase `.active`. Las páginas existentes:

| ID | Descripción |
|----|------------|
| `loading` | Pantalla de carga inicial (animación de barra) |
| `login` | Landing pública + modal de login/registro |
| `list` | Lista de jugadores con filtros y búsqueda (página principal) |
| `form` | Formulario de creación/edición de jugador |
| `detail` | Vista detalle de un jugador |
| `dashboard` | Analytics con Chart.js |
| `compare` | Comparador de hasta 4 jugadores |
| `map` | Mapa interactivo con Leaflet |
| `alerts` | Sistema de alertas y seguimiento |
| `shared` | Vista pública de informe compartido (acceso por URL `?report=ID`) |

### Variables globales principales

```javascript
let currentUser = null;    // objeto user de Firebase Auth
let players = [];          // array de jugadores del usuario actual
let currentId = null;      // ID del jugador actualmente seleccionado
let loginMode = 'login';   // 'login' | 'register'
let unsubscribe = null;    // función para cancelar listener de Firestore (players)
let unsubAlerts = null;    // función para cancelar listener de Firestore (alerts)
let activeFilters = { estado: '' };  // filtros activos en la lista
let alerts = [];           // array de alertas del usuario
let isDemo = false;        // true cuando se usa el modo demo
let formTags = [];         // etiquetas seleccionadas en el formulario
let playerPhoto = '';      // base64 de la foto del jugador en el formulario
let cvClubHistory = [];    // historial de clubes para el Sports CV
let leafletMap = null;     // instancia de Leaflet map
let mapMarkers = [];       // marcadores actuales del mapa
let _charts = {};          // instancias de Chart.js (destruir antes de recrear)
```

---

## Colores corporativos (CSS variables)

```css
--navy:    #0B0F1A   /* Fondo oscuro principal */
--navy2:   #111827   /* Fondo oscuro medio */
--navy3:   #1a2236   /* Fondo oscuro claro */
--purple:  #7C3AED   /* Color principal de marca */
--purple2: #8B5CF6   /* Variante media */
--purple3: #A78BFA   /* Variante clara */
--gold:    #F59E0B   /* Alertas/acento dorado */
--white:   #F8FAFC   /* Blanco base */
--slate:   #64748B   /* Texto secundario */
--slateL:  #94A3B8   /* Texto terciario */
--border:  rgba(255,255,255,.08)  /* Bordes en fondos oscuros */
--borderL: #E2E8F0   /* Bordes en fondos claros */
--bg:      #F4F6FA   /* Fondo de la app (modo claro) */
--text:    #0B0F1A   /* Texto principal */
--text2:   #374151   /* Texto secundario */
--green:   #059669   /* Valores buenos (>=8) */
--red:     #DC2626   /* Valores malos (<6) */
--amber:   #D97706   /* Valores medios (6-7) */
```

### Colores de estados de jugador
- **Activo:** `#34D399` (verde esmeralda)
- **En seguimiento:** `#818CF8` (violeta)
- **En espera:** `#FBBF24` (amarillo)
- **Descartado:** `#F87171` (rojo claro)

### Colores del comparador (hasta 4 jugadores)
```javascript
const CMP_COLORS = ['#7C3AED', '#3B82F6', '#10B981', '#F59E0B'];
```

---

## Estructura de datos en Firestore

### Colección `users/{uid}/players/{playerId}`

Cada documento representa un jugador:

```javascript
{
  id: String,               // timestamp en ms como string (Date.now().toString())
  nombre: String,           // nombre completo (campo obligatorio)
  fecha: String,            // fecha de nacimiento ISO "YYYY-MM-DD"
  posicion: String,         // ej: "Delantero Centro", "Central"
  pieDom: String,           // "Derecho" | "Izquierdo" | "Ambidiestro"
  altura: String,           // en cm, almacenado como string
  peso: String,             // en kg, almacenado como string
  club: String,             // nombre del club actual
  localidad: String,        // localidad donde vive el jugador
  localidadClub: String,    // localidad donde juega el club
  catMain: String,          // "Benjamín"|"Alevín"|"Infantil"|"Cadete"|"Juvenil"|"Sénior"
  catSub: String,           // división dentro de la categoría
  ojeoMain: String,         // categoría para la que se le ojea
  ojeoSub: String,          // división objetivo del ojeo
  estado: String,           // "Activo"|"En seguimiento"|"En espera"|"Descartado"
  tecnica: Number,          // valoración 1-10
  tactica: Number,          // valoración 1-10
  fisico: Number,           // valoración 1-10
  actitud: Number,          // valoración 1-10
  telefono: String,
  contacto: String,         // nombre del contacto (padre/madre/agente)
  relacion: String,         // "Padre/Madre"|"Tutor"|"Representante"|"Otro"
  representante: String,
  interes: String,          // "Desconocido"|"Sin interés"|"Podría valorar"|"Interesado"|"Muy interesado"
  tags: Array<String>,      // etiquetas del jugador
  foto: String,             // base64 JPEG comprimida a 200px max
  notasPrivadas: String,    // notas internas (NO se comparten)
  visitas: Array<{          // historial de visitas a partidos
    id: String,             // timestamp como string
    fecha: String,          // "YYYY-MM-DD"
    partido: String,        // descripción del partido
    nota: String,           // observaciones del scout
    valoracion: Number      // 1-10
  }>,
  cvClubs: Array<{          // trayectoria para Sports CV
    club: String,
    etapa: String,          // ej: "Alevín-Cadete"
    categoria: String       // ej: "Superliga"
  }>,
  updatedAt: String         // ISO timestamp de última modificación
}
```

### Colección `users/{uid}/alerts/{alertId}`

```javascript
{
  id: String,               // timestamp como string
  playerId: String,         // ID del jugador (puede ser vacío)
  date: String,             // fecha límite "YYYY-MM-DD"
  message: String,          // texto de la alerta
  priority: String,         // "info"|"warning"|"urgent"
  done: Boolean,            // si está completada
  createdAt: String,        // ISO timestamp
  doneAt: String            // ISO timestamp (cuando se completó)
}
```

### Colección `shared-reports/{shareId}`

Informes públicos generados con "Compartir por enlace":

```javascript
{
  nombre, posicion, club, catMain, catSub, ojeoMain, ojeoSub,
  pieDom, fecha, altura, peso, estado,
  tecnica, tactica, fisico, actitud,
  tags: Array<String>,
  foto: String,             // base64
  visitas: Array<{fecha, partido, nota, valoracion}>,
  sharedAt: String,         // ISO timestamp
  sharedBy: String          // email del scout
  // Nota: notasPrivadas NO se almacena aquí (seguridad)
}
```

### Colección `config/apikey`

```javascript
{
  key: String,              // clave API de Anthropic
  updatedAt: String         // ISO timestamp
}
```

---

## Subcategorías (SUBCATS)

Las categorías tienen divisiones jerárquicas:

```javascript
const SUBCATS = {
  Benjamín: ['Segunda','Primera','Autonómica','Superliga','División de Honor'],
  Alevín:   ['Segunda','Primera','Autonómica','Superliga','División de Honor'],
  Infantil: ['Segunda','Primera','Autonómica','Superliga','División de Honor'],
  Cadete:   ['Segunda','Primera','Autonómica','Superliga','División de Honor'],
  Juvenil:  ['Segunda','Primera','Autonómica','Nacional','División de Honor'],
  Sénior:   ['2º Autonómica','1º Autonómica','Preferente','3ºRFEF','2ºRFEF','1ºRFEF','2º División','1º División']
};
```

---

## Etiquetas de jugadores (AVAILABLE_TAGS)

```javascript
['Rápido','Líder','Goleador','Creativo','Lesionado','Canterano','Polivalente','Disciplinado']
```

También se permiten etiquetas personalizadas. Cada etiqueta tiene una clase CSS específica (`tag-rápido`, `tag-líder`, etc.) con colores propios.

---

## Funcionalidades implementadas

### 1. Landing page (pública)
- Hero con CTA, sección de problema, features, "cómo funciona", founder quote, pricing (plan gratuito), FAQ con acordeón, footer.
- Nav con links internos y botones de login/demo.

### 2. Autenticación
- Firebase Auth: email/password (login y registro).
- Recuperación de contraseña por email.
- Login via modal overlay que se puede abrir desde la landing.
- Admin email especial (`mariop123478@gmail.com`) con acceso a configuración de API key.

### 3. Modo demo
- 8 jugadores de ejemplo con datos reales de la región de Murcia/España.
- Sin Firebase: los cambios son solo en memoria.
- Funciones de compartir/guardar API key deshabilitadas en demo.
- Banner amarillo permanente indicando que es demo.

### 4. Lista de jugadores
- Búsqueda por texto (nombre, club, posición, etiquetas) con debounce 200ms.
- Filtros: estado (chips), categoría jugador, categoría ojeo, posición, pie, año de nacimiento.
- Ordenación: nombre A-Z/Z-A, mejor/peor nota, más joven/mayor, más visitado, último editado.
- Estadísticas en hero: total, activos, seguimiento, espera, descartados.
- Exportar CSV con BOM UTF-8 para Excel.
- Detección de duplicados al crear jugador.

### 5. Formulario de jugador (4 secciones)
- **01 Ficha técnica:** foto, nombre, fecha nacimiento, posición, pie, altura, peso, club, localidad jugador/club, categoría/división (jugador + ojeo), estado, etiquetas.
- **02 Valoración scout:** sliders 1-10 para técnica, táctica, físico, actitud. Nota global calculada automáticamente.
- **03 Contacto y entorno:** nombre contacto, relación, teléfono, representante, interés en cambio de club.
- **04 Notas privadas:** textarea marcada como confidencial, no aparece en informes compartidos.

### 6. Detalle de jugador
- Header dark con avatar, nombre, nota global y chips de info.
- Rating boxes (4 dimensiones) + radar chart SVG.
- Secciones: etiquetas, ficha técnica, contacto, notas privadas (solo para el scout), evolución.
- Historial de visitas: cada visita muestra fecha, partido, valoración coloreada y nota.
- Cada visita tiene botón de informe de partido individual (PDF) y botón de eliminar.
- Gráfico de evolución (Chart.js line) con línea de tendencia (regresión lineal) — se muestra con ≥2 visitas con valoración.
- Análisis de evolución con IA.
- Botones: Editar, Informe PDF, Sports CV, Compartir.

### 7. Informes PDF
Todos los informes se generan en una nueva ventana con HTML completo e incluyen botón "Guardar como PDF" (window.print() con @media print).

**Informe global:** Análisis completo con IA (Claude API) o fallback algorítmico. Incluye header dark, valoraciones, radar SVG, ficha técnica, análisis del scout y historial de visitas con timeline.

**Informe de partido individual:** Informe por visita específica. Header dark con valoración de ese partido, datos del partido, observaciones del scout, valoraciones globales del jugador.

### 8. Sports CV
- Popup modal para ingresar trayectoria en clubes (club, etapa, categoría) y descripción libre.
- La IA pule la descripción (o se usa tal cual si no hay API key).
- CV con diseño profesional: foto/avatar, datos personales, radar SVG, barras de valoración, trayectoria timeline y descripción.
- La trayectoria se guarda en `player.cvClubs` en Firestore.

### 9. Compartir por enlace
- Genera un documento en `shared-reports/{shareId}` en Firestore.
- URL: `?report={shareId}` — cualquier persona puede verlo sin cuenta.
- La vista compartida muestra: hero dark, radar, valoraciones, ficha técnica (sin notas privadas).
- No disponible en modo demo.

### 10. Dashboard Analytics
7 gráficos con Chart.js:
- Doughnut: distribución por estado
- Bar horizontal: distribución por posición
- Bar: jugadores por categoría (con colores por categoría)
- Radar: media global de valoraciones
- Lista top 10 jugadores por nota (con medallas gold/silver/bronze)
- Doughnut: pie dominante
- Bar horizontal: etiquetas más usadas

### 11. Comparador (hasta 4 jugadores)
- Slots dinámicos (empiezan en 2, máximo 4 con botón "+").
- Preview con foto/avatar de cada jugador seleccionado.
- **2 jugadores:** Barras "duel" con efecto glow (ganador brillante).
- **3-4 jugadores:** Barras segmentadas.
- Radar multi-dataset con colores distintos.
- Tabla de datos comparativa.
- Recomendación IA: elige al mejor y justifica (con fallback algorítmico).

### 12. Mapa interactivo
- Leaflet + OpenStreetMap.
- Base de datos interna de ~100 ciudades españolas con coordenadas.
- Filtro: localidad del jugador vs localidad del club.
- Marcadores circulares con tamaño proporcional al número de jugadores.
- Color del marcador según estado del primer jugador del grupo.
- Popup con lista de jugadores en esa localidad (clic para ir al detalle).
- Stats: jugadores en mapa, localidades, jugadores sin localidad.

### 13. Alertas y seguimiento
- CRUD completo de alertas con Firestore.
- Prioridades: info (azul), warning (amarillo), urgent (rojo).
- Alertas vencidas se resaltan en rojo.
- Acciones: completar, posponer 7 días, eliminar.
- Badge con contador en el botón del header.
- Las alertas pueden vincularse a un jugador específico.

### 14. PWA
- Manifest dinámico generado via Blob URL.
- Service Worker minimal (install prompt + offline fallback).
- Meta tags para iOS (apple-mobile-web-app-*).
- Icono SVG inline como favicon y apple-touch-icon.

---

## Integración con IA (Claude API)

La API key de Anthropic se almacena en:
1. Firestore `config/apikey` (principal, acceso solo admin)
2. `localStorage` clave `sct_apikey_v1` (caché local)

**Endpoint:** `https://api.anthropic.com/v1/messages`
**Headers especiales:** `anthropic-dangerous-direct-browser-access: true` (llamada directa desde browser)
**Modelo:** `claude-sonnet-4-20250514`

### Usos de IA:
1. **Informe global** — 3 párrafos: perfil/estilo, fortalezas, áreas de mejora. 220-270 palabras.
2. **Sports CV description** — Pule la descripción libre del scout (máx 80 palabras).
3. **Análisis de evolución** — 3 frases: cómo era antes, cómo ha cambiado, tendencia/recomendación.
4. **Recomendación comparador** — 3 frases: a quién elegir, ventaja, riesgo.

Todos tienen fallback algorítmico (`generateSmartSummary`, `generateSmartEvolution`, `generateSmartRecommendation`).

---

## Funciones utilitarias clave

```javascript
esc(s)         // XSS: escapa HTML via div.textContent
ini(nombre)    // Iniciales del nombre (2 letras mayúsculas)
avg(player)    // Nota global: media de técnica+táctica+físico+actitud
rateColor(v)   // Color según nota: verde>=8, amber>=6, rojo<6
badgeClass(e)  // Clase CSS del badge de estado
formatDate(d)  // "YYYY-MM-DD" → "15 de enero de 2025"
getYear(d)     // Extrae el año de nacimiento
todayISO()     // "YYYY-MM-DD" de hoy
todayStr()     // "15 de enero de 2025"
showPage(p)    // Navega entre páginas (quita .active, añade en #page-p)
showToast(m)   // Toast notification bottom-right (2.2s)
showSavePill() // Pill "✓ Guardado" en el header (2s)
showConfirm()  // Dialog de confirmación reutilizable
isAdmin()      // true si currentUser.email === ADMIN_EMAIL
getApiKey()    // Lee la API key de localStorage
```

---

## Convenciones de código

### Exposición al HTML
Las funciones llamadas desde atributos HTML (`onclick="..."`) se exponen en `window`:
```javascript
window.openDetail = function(id) { ... };
window.toggleFilter = function(el) { ... };
// etc.
```

### IDs de elementos HTML
- Formulario: `f-{campo}` (ej: `f-nombre`, `f-tecnica`)
- Sliders: `f-{campo}` + display en `v-{campo}` (ej: `f-tecnica` + `v-tecnica`)
- Detalle: `d-{campo}` (ej: `d-nombre`, `d-nota`, `d-tec`)
- Estadísticas header: `st-{estado}` (ej: `st-total`, `st-activos`)
- Charts: `chart-{nombre}` (ej: `chart-estado`, `chart-evolucion`)
- Comparador: `cmp-p{i}` para selects, `cmp-prev{i}` para previews

### Chart.js — destroy antes de recrear
Siempre usar `destroyChart(id)` antes de crear un chart nuevo para evitar memory leaks:
```javascript
function destroyChart(id) {
  if (_charts[id]) { _charts[id].destroy(); delete _charts[id]; }
}
```

### Persistencia con Firestore
```javascript
// Guardar jugador
async function persistPlayer(p) {
  const { id, ...data } = p;
  await setDoc(doc(db, 'users', currentUser.uid, 'players', id), data);
}

// Eliminar jugador
async function removePlayer(id) {
  await deleteDoc(doc(db, 'users', currentUser.uid, 'players', id));
}
```

### Sync en tiempo real
`onSnapshot` para players y alerts. Los unsubscribers se guardan en `unsubscribe` y `unsubAlerts` y se cancelan al hacer logout.

### Modo demo vs real
Siempre verificar `if (isDemo)` antes de llamar a Firebase:
```javascript
if (isDemo) {
  // manipular array local players[]
  return;
}
// llamada a Firebase...
```

### Protección XSS
Siempre usar `esc(valor)` al renderizar HTML dinámico. Nunca concatenar strings de usuario directamente en innerHTML.

### Fotos de jugadores
Se comprimen a máximo 200px y calidad JPEG 70% via Canvas API. Se almacenan como base64 en Firestore directamente en el documento del jugador. Límite visual: 2MB de archivo original.

---

## Radar Chart (SVG)

El radar se dibuja manualmente con SVG (sin Chart.js). Función `renderRadar(containerId, scores)`:
- 4 dimensiones: técnica, táctica, físico, actitud
- Grid de 5 niveles concéntricos (polígonos)
- Vértice superior = técnica, sentido horario
- Puntos de datos en púrpura `#7C3AED` con valor numérico sobre cada punto
- Fill semi-transparente `rgba(124,58,237,0.15)`

El mismo algoritmo se replica inline en los informes PDF y en la vista compartida (sin llamar a `renderRadar` porque necesita generarse como string SVG).

---

## Mapa — CITY_COORDS

Base de datos interna con coordenadas de ~100 ciudades españolas. El matching se hace comparando el texto de la localidad (lowercase) con las claves del objeto. Incluye especial cobertura de la Región de Murcia.

La función de matching no es exacta (usa `includes`), por lo que hay que tener cuidado con localidades ambiguas.

---

## Responsive design

Breakpoints:
- `≤768px`: columna única en formularios, stats en 3 columnas, menú simplificado, datos secundarios ocultos en lista.
- `≤480px`: stats en 2 columnas (el total ocupa ancho completo), avatares más pequeños.

En móvil, la lista de jugadores solo muestra nombre, nota y estado (se ocultan sub-info, pie, año con `display:none`).

---

## Seguridad

- **XSS:** función `esc()` en todo HTML dinámico.
- **Firestore rules:** cada usuario solo accede a su subcolección `users/{uid}/`.
- **Notas privadas:** nunca se escriben en `shared-reports` ni en informes compartidos.
- **API key:** solo el admin puede configurarla. Se guarda en Firestore y localStorage, nunca en el HTML.
- **Fotos:** base64 inline (no Firebase Storage), comprimidas para reducir tamaño.

---

## Firebase config

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyAZjsB6kk96ZYa685ITb-hW5uycvBYfQeY",
  authDomain: "mm-talent-scouting.firebaseapp.com",
  projectId: "mm-talent-scouting",
  storageBucket: "mm-talent-scouting.firebasestorage.app",
  messagingSenderId: "569472528879",
  appId: "1:569472528879:web:36b44fa500d49db3b6861a"
};
```

Firebase SDK se importa desde CDN via ES modules (`https://www.gstatic.com/firebasejs/10.12.0/...`).

---

## Flujo de autenticación

1. `onAuthStateChanged` se ejecuta al cargar.
2. Primero verifica si la URL tiene `?report=ID` (informe compartido) — si sí, muestra vista pública.
3. Si hay usuario: inicia sync con Firestore, muestra header y lista.
4. Si no hay usuario y no es demo: muestra landing/login.
5. Demo: no requiere auth, usa datos locales en memoria.

---

## Exportación CSV

Genera un CSV con BOM (`\uFEFF`) para compatibilidad con Excel en español. Separador: punto y coma (`;`). Incluye todos los campos del jugador excepto foto, notas privadas y visitas detalladas (solo el conteo).

---

## Datos de demo

8 jugadores de la zona de Murcia/España con visitas, etiquetas y datos realistas. Los datos están hardcodeados en `DEMO_PLAYERS`. En modo demo, los cambios se aplican en memoria (array `players`), pero no persisten al recargar.
