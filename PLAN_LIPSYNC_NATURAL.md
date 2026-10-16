# PLAN: Lip-sync y expresiones naturales para el avatar "Amelia" (web)

> **Para la IA que continúe este trabajo:** lee este archivo completo antes de tocar código.
> Al terminar cada paso, marca la casilla `[x]` y agrega una línea en la **Bitácora** (al final).
> No modifiques `opcion1_*.html`, `opcion2_*.html`, `opcion3_*.html` ni `index.html`: son referencias.
> Todo el trabajo nuevo va en **`opcion4_lipsync_natural.html`**.

---

## 0. Contexto del proyecto

- **Ruta:** `d:\Sistole\Sistole_Avatar\`
- **Stack:** HTML + CSS + JS vanilla en un solo archivo, sin build. Se abre directo en el navegador o con un servidor estático.
- **Voz:** `window.speechSynthesis`, que busca una voz femenina latina.
- **Cerebro:** API de Gemini (`consultarGemini`) con respuestas cortas. **Esta lógica no se toca**: se copia tal cual.
- **Imágenes** (`Img/`, 848×1264 px, fotorrealistas, mismo encuadre y pose):
  - `idle_hd.jpg`: boca cerrada
  - `talking_mid.jpg`: boca entreabierta
  - `talking_hd.jpg`: boca abierta
- **Coordenadas aproximadas en la imagen original (848×1264).** Hay que verificarlas en el Paso 2:
  - Boca: **x≈441, y≈333** (52 %, 26.3 %)
  - Ojos: izquierdo ≈(417, 285), derecho ≈(468, 285)
  - Ancho de la cara ≈110 px; la cabeza va de y≈180 a y≈360

### Opciones existentes (diagnóstico)
| Archivo | Técnica | Problema |
|---|---|---|
| `opcion1_corte_instantaneo.html` | Cambia la foto completa | Todo el frame "salta" |
| `opcion2_recorte_boca.html` | `clip-path: ellipse(40% 25% at 50% 35%)` | Máscara enorme (cubre el torso) y bordes duros |
| `opcion3_canvas_suave.html` | Canvas a 60 FPS, lerp de opacidad | "Boca fantasma" (dos bocas mezcladas) y máscara enorme |

**Fallas comunes que este plan corrige:**
1. La secuencia de visemas es fija y aleatoria cada 90 ms; no depende del texto.
2. La duración se estima con `texto.length * 80`; `utterance.onend` no detiene la boca.
3. La máscara cubre del 10 al 60 % de la altura, cuando la boca está en el 26 %.
4. La cara mide unos 50 px en pantalla, así que casi no se ve.
5. No hay parpadeo, respiración ni micro-movimientos.

---

## 1. Objetivo del entregable (Nivel 1)

Crear `opcion4_lipsync_natural.html`, basado en la **opción 3 (Canvas)**, con:
- Encuadre de busto y zoom suave al hablar
- Máscara de boca pequeña y difuminada (sin bordes ni fantasma)
- Apertura de mandíbula continua (`jawOpen` 0–1) con física de resorte
- Visemas derivados del texto en español, sincronizados con `onboundary` y `onend`
- Parpadeo, respiración y micro-movimiento de cabeza
- Pequeños gestos de énfasis según la puntuación

---

## 2. Pasos

### Paso 1 – Crear el archivo base
- [x] Copiar `opcion3_canvas_suave.html` como `opcion4_lipsync_natural.html`.
- [x] Cambiar `<title>` y `<h1>` a "Opción 4: Lip-sync Natural".
- [x] Mantener intactos: formulario, `seleccionarVozFemeninaLatina`, `getActiveApiKey`, `cambiarApiKey`, `consultarGemini` y el listener `submit`.

**Criterio de aceptación:** el archivo abre y funciona igual que la opción 3.

### Paso 2 – Constantes de geometría y overlay de depuración
- [x] Definir arriba del script, en coordenadas de la imagen original:
```js
const SRC_W = 848, SRC_H = 1264;
const MOUTH = { x: 441, y: 333, rx: 34, ry: 26 };   // radio útil de la máscara
const EYES  = [{ x: 417, y: 285 }, { x: 468, y: 285 }];
const HEAD  = { x: 441, y: 270, rx: 75, ry: 105 };
const DEBUG = new URLSearchParams(location.search).has('debug');
```
- [x] Si `DEBUG` está activo, dibujar círculos rojos o verdes sobre `MOUTH`, `EYES` y `HEAD` para calibrar. Abrir `opcion4_lipsync_natural.html?debug`.
- [x] Ajustar los valores hasta que coincidan con la cara. Comparar también `talking_hd.jpg`: la boca abierta debe quedar dentro de la máscara.

> ✅ **Valores calibrados (en el código):** `MOUTH {442,335,rx32,ry26}`, ojos `{418,281}` y `{468,285}` (rx13, ry6), `HEAD {442,272,rx74,ry112}`, `MID_ALIGN dy=+2.5` (`talking_mid.jpg` está ~2.5 px más arriba). En modo `?debug`, al hacer clic en el canvas se muestra la coordenada original para afinar.

**Criterio de aceptación:** los marcadores caen sobre la boca y los ojos reales.

### Paso 3 – Encuadre de busto y cámara
- [x] No dibujar la imagen completa. Usar `drawImage(img, sx, sy, sw, sh, 0, 0, canvas.width, canvas.height)` con una "cámara":
```js
// Busto: proporción 2:3 igual al contenedor 360x540
const CAM_IDLE = { cx: 441, cy: 390, w: 320 };   // h = w * 1.5
const CAM_TALK = { cx: 441, cy: 360, w: 290 };   // zoom leve al hablar
let cam = { ...CAM_IDLE };                       // se interpola en cada frame (lerp 0.03)
```
- [x] Crear una función `srcToCanvas(x, y)` que convierta coordenadas de la imagen a coordenadas del canvas según la cámara. La máscara y el overlay de depuración la usarán.
- [x] Opcional: un botón "Plano completo / Busto" para alternar entre encuadres.

**Criterio de aceptación:** la cara se ve grande (≥120 px de ancho en pantalla) y el zoom al hablar es suave.

### Paso 4 – Máscara difuminada (adiós bordes y fantasma)
- [x] Crear un canvas fuera de pantalla `mouthLayer` del tamaño del canvas principal.
- [x] En cada frame, para la imagen de boca que corresponda:
```js
mctx.clearRect(0,0,W,H);
mctx.globalCompositeOperation = 'source-over';
mctx.drawImage(imgBoca, sx, sy, sw, sh, 0, 0, W, H);   // misma cámara
mctx.globalCompositeOperation = 'destination-in';
const p = srcToCanvas(MOUTH.x, MOUTH.y), r = MOUTH.rx * escala;
const g = mctx.createRadialGradient(p.x, p.y, r*0.45, p.x, p.y, r);
g.addColorStop(0, 'rgba(0,0,0,1)'); g.addColorStop(1, 'rgba(0,0,0,0)');
mctx.fillStyle = g;
mctx.save(); mctx.translate(p.x, p.y); mctx.scale(1, MOUTH.ry/MOUTH.rx);
mctx.translate(-p.x, -p.y); mctx.fillRect(0,0,W,H); mctx.restore();
ctx.globalAlpha = alpha; ctx.drawImage(mouthLayer, 0, 0); ctx.globalAlpha = 1;
```
- [x] **Anti-fantasma:** mezclar solo dos imágenes **adyacentes** según `jawOpen` (ver Paso 5). Nunca superponer `mid` y `high` sobre `idle` al mismo tiempo con alfas parciales.

**Criterio de aceptación:** no se ve ningún borde de elipse y en ningún momento hay dos bocas visibles.

### Paso 5 – Mandíbula continua con resorte
- [x] Reemplazar `mouthState`, `midAlpha` y `highAlpha` por:
```js
let jawTarget = 0, jawOpen = 0, jawVel = 0;
// en drawFrame:
jawVel += (jawTarget - jawOpen) * 0.35;
jawVel *= 0.55;
jawOpen = Math.min(1, Math.max(0, jawOpen + jawVel));
```
- [x] Mapeo a imágenes (mezcla de pares adyacentes):
  - `jawOpen` 0 → 0.5: base `idle` + `mid` con alfa `jawOpen/0.5`
  - `jawOpen` 0.5 → 1: base `mid` (con máscara, alfa 1) + `high` con alfa `(jawOpen-0.5)/0.5`

**Criterio de aceptación:** la boca abre y cierra con inercia, sin parpadeos de imagen.

### Paso 6 – Texto → visemas en español
- [x] Crear `textoAVisemas(texto)`, que devuelve un array `[{t: ms, open: 0..1}]`:
```js
const APERTURA = {
  a:1.0, á:1.0, o:0.85, ó:0.85, e:0.6, é:0.6, i:0.4, í:0.4, u:0.45, ú:0.45,
  m:0, b:0, p:0,            // labios cerrados (¡importante para naturalidad!)
  f:0.15, v:0.1, l:0.35, d:0.3, t:0.3, n:0.3, s:0.25, r:0.35, c:0.3, g:0.3, j:0.35, y:0.35, ll:0.35, ñ:0.3
};
// Duración base por letra ≈ 65 ms / utterance.rate
// Espacio: 40 ms con open 0.1 | ',' : 180 ms open 0 | '.', '?', '!': 350 ms open 0
// La h es muda (se omite). Los dígitos se expanden aproximadamente (3 letras c/u).
```
- [x] Bajar la tasa de las vocales consecutivas: no se cierra la boca entre "ue", "ia", etc.

**Criterio de aceptación:** "mamá papá" muestra cierres claros en m/p; "hola" abre en la o y en la a.

### Paso 7 – Sincronización con la voz real
- [x] Crear un scheduler en `drawFrame` que lea la línea de tiempo y asigne `jawTarget` según `performance.now() - inicioSegmento`.
- [x] **`utterance.onstart`:** iniciar la línea de tiempo, agregar la clase `talking` y activar `CAM_TALK`.
- [x] **`utterance.onboundary`** (`e.name === 'word'`): **re-sincronizar**. Saltar al índice de la línea de tiempo que corresponde a `e.charIndex` (guardar `charIndex` en cada entrada del array).
- [x] **Fallback:** algunas voces (por ejemplo las de red de Google en Chrome) **no emiten `onboundary`**. En ese caso la línea de tiempo corre sola con la duración estimada y se escala según `utterance.rate`.
- [x] **`utterance.onend` / `onerror`:** `jawTarget = 0`, quitar la clase `talking`, volver a `CAM_IDLE` y rehabilitar el input.
- [x] Eliminar el `setInterval` de 90 ms y el `setTimeout` de `texto.length * 80`. Dejar solo un timeout de seguridad de `duraciónEstimada * 1.5` por si `onend` nunca llega (bug conocido de Chrome con textos largos).
- [x] Opcional (bug de Chrome que corta la voz a ~15 s): dividir el texto en oraciones y encolar varios `SpeechSynthesisUtterance`.

**Criterio de aceptación:** la boca se detiene en menos de 150 ms después de que la voz termina, y las pausas en las comas se ven.

### Paso 8 – Vida en reposo
- [x] **Respiración:** escala global de la cámara `1 + sin(t*0.0013)*0.004` y desplazamiento vertical de ±1 px.
- [x] **Micro-movimiento de cabeza:** ruido suave (suma de 2–3 senos con frecuencias irracionales). Dibujar la región `HEAD` con una máscara difuminada (misma técnica del Paso 4), desplazada ±1.5 px y rotada ±0.6°. Amplitud ×2 mientras habla.
- [x] **Parpadeo:**
  - Opción A (recomendada): generar el asset `Img/blink.jpg`, una copia de `idle_hd.jpg` con los ojos cerrados (editar la imagen con IA, mismo encuadre), y mezclarlo con máscara difuminada sobre `EYES`.
  - Opción B (sin asset): comprimir verticalmente la franja de cada ojo con `drawImage` (escala Y 1→0.1→1) y cubrir con el color de piel muestreado sobre el ojo.
  - Intervalo aleatorio de 2–6 s, duración de unos 140 ms (cerrar 60 ms, abrir 80 ms) y 15 % de probabilidad de doble parpadeo.

**Criterio de aceptación:** en reposo el avatar se ve vivo, pero sin mareo ni costuras visibles en la máscara de la cabeza.

### Paso 9 – Gestos de énfasis
- [x] Con `?`: inclinación leve de cabeza (+0.8°) durante la última palabra.
- [x] Con `!` o con palabras en MAYÚSCULA: "nod" (cabeceo) de 2 px hacia abajo y luego arriba en 250 ms.
- [x] Al inicio de cada oración: micro-zoom de la cámara (+1 %) que decae.

**Criterio de aceptación:** los gestos son sutiles y coherentes con la frase.

### Paso 10 – Pruebas y cierre
> ✅ **Validación del usuario (2026-10-16):** la gesticulación de la boca, el acercamiento al hablar y el parpadeo se ven bien y naturales. El usuario indica que "hay cosas que mejorar"; ver la sección **2.1 Ronda de ajustes**.
> Servidor local: `npx -y http-server -p 8765 -c-1` en la carpeta del proyecto → abrir `http://localhost:8765/opcion4_lipsync_natural.html?debug`
- [x] Probar en el navegador del usuario (validado visualmente por el usuario).
- [x] Frases de prueba:
  - "Hola, me llamo Amelia. ¿En qué puedo ayudarte hoy?"
  - "Mamá, papá, bebé." (cierres de labios)
  - "¡Excelente! Tu cita está confirmada."
- [ ] Revisar en la consola que no haya errores y que el FPS sea estable (DevTools > Performance).
- [ ] Actualizar `index.html` con un enlace a la opción 4, **solo si el usuario lo pide**.

### 2.1 Ronda de ajustes (pendiente de que el usuario los indique)
- [ ] _(Anotar aquí cada mejora que pida el usuario, con su estado.)_

---

## 3. Fases futuras (no hacer sin aprobación del usuario)

- **Nivel 2a:** generar sprites de visemas (A, E, I, O, U, MBP, FV, L, reposo) y cejas arriba como PNG con alfa.
- **Nivel 2b:** deformación de malla con WebGL (PixiJS `MeshPlane` o three.js). Se usan landmarks de MediaPipe Face Landmarker calculados una sola vez sobre `idle_hd.jpg` y se deforman la mandíbula, las comisuras, los párpados y las cejas.
- **Nivel 2c:** TTS con visemas. Azure Speech SDK tiene el evento `visemeReceived`; también Google Cloud TTS con `<mark>`, o ElevenLabs con timestamps. Otra vía es Web Audio `AnalyserNode` (librería `wawa-lipsync`).
- **Nivel 3:** Live2D Cubism Web SDK, avatar 3D VRM/TalkingHead, o avatar de video IA (HeyGen / D-ID / Simli).

> ⚠️ **Seguridad:** las API keys de Gemini están en el código (base64). En producción deben moverse a un backend o proxy.

---

## 4. Bitácora de progreso

| Fecha | IA / Persona | Paso | Nota |
|---|---|---|---|
| 2026-10-16 | Claude | — | Análisis de las 3 opciones y creación de este plan. Aún no se ha escrito código. |
| 2026-10-16 | Claude | 1–9 | Creado `opcion4_lipsync_natural.html` con encuadre de busto y zoom, máscaras difuminadas (`destination-in`), mandíbula con resorte a 60 Hz fijos, visemas en español, sincronización con `onstart`/`onboundary`/`onend` y adaptación de velocidad, frases encoladas, simulación si no hay voz, parpadeo procedural (estira la piel del párpado), respiración, micro-movimiento de cabeza en una capa difuminada, gestos con `?`, `!` y MAYÚSCULAS, y overlay `?debug`. |
| 2026-10-16 | Claude | 10 | Sintaxis y visemas validados en Node. **Pendiente:** validación visual (el navegador de la IA falló). Posibles ajustes: radios de la máscara, `LID_DY` del párpado y amplitud del movimiento de cabeza. |
| 2026-10-16 | Usuario | 10 | Validación visual OK: le gusta la gesticulación, el zoom al hablar y el parpadeo natural. Quedan mejoras por definir (sección 2.1). |
