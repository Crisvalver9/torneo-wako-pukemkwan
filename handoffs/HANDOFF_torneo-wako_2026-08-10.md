# HANDOFF — Torneo WAKO Pukem Kwan → continuar en Claude Code

## Cómo usar este handoff

1. Descarga **dos archivos** desde el chat de claude.ai (los mismos que Claude te presentó ahí): `torneo-wako-pukemkwan.html` y este mismo `HANDOFF_torneo-wako_2026-08-10.md`.
2. Crea una carpeta local (ej. `torneo-wako-pukemkwan/`) y pon ambos archivos adentro.
3. Abre **Claude Code** apuntando a esa carpeta (terminal: `cd torneo-wako-pukemkwan && claude`, o desde la app de escritorio de Claude eligiendo esa carpeta).
4. Pega el **PROMPT DE ARRANQUE** que está al final de este documento como tu primer mensaje.

A diferencia del chat normal, Claude Code corre en tu computador de verdad: puede crear archivos, correr `git`, e interactuar con GitHub directamente (usando tus credenciales locales), sin que tengas que copiar/pegar cada paso.

---

## Contexto del proyecto

App de torneo para la **Academia Taekwon-Do ITF Pukem Kwan** (Castro, Chiloé — instructor Marcelo Vergara), para gestionar un torneo interno de **kickboxing WAKO** (Point Fighting y Light Contact). Es una sola página HTML autocontenida (sin build step, JS vanilla, sin frameworks) pensada para usarse desde el celular durante el torneo: inscripción de luchadores, generación de llaves, y marcador en vivo sincronizado entre varios dispositivos (jueces, jefe de mesa, pantalla pública).

## Qué se hizo hasta ahora

- **`torneo-wako-pukemkwan.html`**: la app completa, funcional, probada con tests de lógica (bracket, mayoría de jueces, escalada de faltas — ver sección de decisiones). Actualmente usa `window.storage`, una API que **solo existe dentro del entorno de artifacts de Claude.ai** — por eso funciona ahí pero no si se abre el archivo suelto (`file://`) ni si se sube a un hosting normal como GitHub Pages tal cual.
- Se investigó el **reglamento oficial WAKO** (Chapter 2 — Tatami Rules, vía wakousa.org) para las divisiones de edad, categorías de peso por sexo, y estructura de asaltos. Estos datos ya están correctamente cargados en el código (constantes `WAKO_DIVISIONS`, `WAKO_WEIGHTS`, `WAKO_DEFAULT_ROUNDS`).
- Se decidió migrar el almacenamiento a **Firebase Realtime Database** (gratis) para poder alojar la app de forma independiente en **GitHub Pages**, con URL propia del usuario (no atada a una cuenta de Claude).

## Decisiones ya tomadas (no volver a preguntar)

- **Arquitectura**: un solo archivo HTML, JS vanilla (sin React ni build step), CSS propio (tema oscuro rojo/azul). Mantener así — no migrar a framework.
- **Sin códigos de combate**: toda la data vive en un único documento compartido (antes `torneo:pukemkwan` en window.storage; en Firebase será la raíz del proyecto, path `/torneo`).
- **Roles de la app** (4, ya implementados como vistas separadas):
  - **Mesa**: crear categorías (división WAKO + sexo + peso + cinturón color/negro + modalidad PF/LC + N° jueces + N° asaltos + duración), inscribir luchadores en vivo, cerrar inscripciones → genera llave de 16 (con byes automáticos), declarar ganador manual o iniciar marcador en vivo.
  - **Jefe de mesa**: control de combates en vivo — iniciar/pausar/reiniciar reloj, +10s/+30s/-10s/-30s, "siguiente asalto" (el puntaje se mantiene entre asaltos, no se resetea — así es la regla WAKO real), "+ falta" con escalada automática (2ª y 3ª falta = -1 punto, 4ª = descalificación automática y cierra el combate). Esto está separado del rol Juez a propósito: en el reglamento WAKO real, los jueces solo anotan puntos, el Central Referee/mesa administra tiempo y sanciones.
  - **Juez**: solo vota Rojo/Azul. Motor de puntuación: Point Fighting = mayoría de jueces (mínimo 2, ventana de ~3.5s desde el primer voto, empate exacto = punto compartido); Light Contact = cada juez suma independiente, se totaliza.
  - **Pantalla pública**: solo lectura (sin botones interactivos, para proyectar sin riesgo). Muestra nombre + escuela/club de cada luchador, categoría completa, marcador, cronómetro, asalto actual, faltas (con conteo numérico), y en Light Contact el desglose por juez (J1, J2, J3), imitando el sistema oficial WAKO.
- **Bracket**: hasta 16 luchadores, siempre Octavos→Cuartos→Semis→3°/4° puesto→Final (nunca cambia de ronda inicial aunque haya pocos inscritos — los espacios vacíos son bye en Octavos). La lógica de "bye vs. esperando ronda anterior" fue una fuente de bugs ya corregida y testeada (ver `buildBracket`/`leafMatch`/`nextRoundMatch` en el código) — **no reescribir esta lógica sin volver a testear los casos de bye/waiting**.
- **Categorías WAKO oficiales** ya cargadas (Children, Younger Cadets, Older Cadets, Juniors, Seniors, Masters/Veterans) con sus pesos exactos por sexo, sacadas del reglamento real (Chapter 2, Article 3). El cinturón color/negro es un criterio del club, no de WAKO — dejarlo así, no presentarlo como oficial.
- **Seguridad de Firebase**: se optó por reglas abiertas permanentes (`{"rules": {".read": true, ".write": true}}`) en vez del modo de prueba de 30 días, porque es solo para gestionar un torneo interno de bajo riesgo. El usuario ya fue informado de este tradeoff.
- **Diseño mobile-first para iPhone**: ya tiene meta tags de `apple-mobile-web-app-capable`, `viewport-fit=cover`, y padding con `env(safe-area-inset-*)` para notch/home indicator. No hace falta una versión "aparte" para iPhone — es el mismo archivo.

## Estado AHORA MISMO

- El usuario **aún no ha creado el proyecto Firebase**. Los pasos que se le dieron (y que Claude Code debe guiar si no los ha hecho):
  1. console.firebase.google.com → crear proyecto (sin Analytics).
  2. Build → Realtime Database → Create Database → test mode.
  3. Pestaña Rules → reemplazar por `{"rules": {".read": true, ".write": true}}` → Publish.
  4. Copiar la URL de la base de datos (algo como `https://PROJECT-default-rtdb.REGION.firebasedatabase.app`).
- **El código NO ha sido migrado todavía** de `window.storage` a Firebase. Hay exactamente 7 líneas que llaman a `window.storage` y necesitan reemplazo (ver "Próximo paso inmediato").
- No se ha creado ningún repo de GitHub todavía.

## Próximo paso inmediato

1. Si el usuario no tiene la URL de Firebase todavía, guiarlo a obtenerla (pasos arriba) antes de tocar código.
2. En `torneo-wako-pukemkwan.html`, reemplazar el storage layer:
   - Agregar cerca del inicio: `const FIREBASE_URL = 'https://<LA_URL_DEL_USUARIO>';` (sin barra final) y `const TORNEO_PATH = FIREBASE_URL + '/torneo.json';`.
   - `loadTorneo()`: cambiar `await window.storage.get(TORNEO_KEY, true)` → `const res = await fetch(TORNEO_PATH); const data = await res.json(); state.torneo = data || emptyTorneo();`
   - `writeTorneo(next)`: cambiar `await window.storage.set(TORNEO_KEY, JSON.stringify(next), true)` → `await fetch(TORNEO_PATH, {method:'PUT', headers:{'Content-Type':'application/json'}, body: JSON.stringify(next)})`.
   - Dentro de `setupTimersForView()` (el `resolveTimer` que resuelve mayorías de Point Fighting) y dentro de `vote()`: mismo patrón, reemplazar las llamadas directas a `window.storage.get(TORNEO_KEY, true)` por `fetch(TORNEO_PATH).then(r=>r.json())`.
   - `saveSession`/`restoreSession` (guardan qué rol/categoría/juez eligió el dispositivo): estos NO necesitan Firebase — son datos personales del dispositivo. Cambiarlos a `localStorage.setItem('wako_session', JSON.stringify(next))` / `localStorage.getItem('wako_session')`, ya no hace falta que sean `async` (localStorage es síncrono). **Ojo**: `localStorage` está prohibido dentro de artifacts de Claude.ai, pero este archivo ya no vive ahí — corre como página web normal, así que aquí sí es correcto usarlo.
3. Probar localmente (`python3 -m http.server` o similar) abriendo el archivo en dos pestañas del navegador y confirmando que el marcador se sincroniza entre ambas.
4. Crear repo de GitHub (con `git`/`gh` CLI si están disponibles) y activar GitHub Pages (Settings → Pages → branch `main` / carpeta raíz). Si el archivo no se llama `index.html`, renombrarlo o configurar Pages para servir ese archivo específico.
5. Entregar al usuario el link final de GitHub Pages.

## Reglas de trabajo con el usuario

- Habla en español (Chile), tono directo y práctico — el usuario es estudiante de Ingeniería Civil Industrial, no programador, así que explica en términos simples qué se está haciendo y por qué, sin asumir jerga de desarrollo.
- No inventes URLs, claves ni credenciales — si falta la URL de Firebase, pregúntala antes de escribir el código de conexión.
- No subas nada a un repo público de GitHub sin que el usuario confirme que quiere que sea público (podría preferir privado, en cuyo caso GitHub Pages requiere plan de pago para repos privados — avisarle de esto si aplica).
- Antes de hacer `git push` o crear el repo remoto, confirma con el usuario.
- Antes de cambiar la lógica del bracket (bye/waiting) o del motor de puntuación (mayoría PF / suma LC / escalada de faltas), vuelve a correr pruebas equivalentes a las descritas arriba — esa lógica ya fue debuggeada con casos reales y es fácil reintroducir bugs sutiles (ver ejemplo: un bye en una ronda posterior NO debe confundirse con "ronda anterior aún pendiente").

## Archivos clave

- `torneo-wako-pukemkwan.html` — la app completa (única fuente de verdad, no hay otros archivos de código).
- Este handoff.

## PROMPT DE ARRANQUE

```
Lee el archivo HANDOFF_torneo-wako_2026-08-10.md en esta carpeta y sigue desde la sección
"Próximo paso inmediato". Es una app de torneo de kickboxing (un solo HTML) que hay que migrar
de window.storage (API de Claude.ai) a Firebase Realtime Database, y luego publicar en GitHub
Pages. Pregúntame por la URL de Firebase si no la tienes. Ve explicándome cada paso en simple,
no soy programador.
```
