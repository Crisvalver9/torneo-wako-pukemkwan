# HANDOFF — Torneo WAKO Pukem Kwan → continuar en otro hilo

## Cómo usar este handoff

1. Abre un chat nuevo en Claude Code, con el modelo que quieras usar, apuntando a esta misma carpeta
   (`C:\Users\luzug\OneDrive\Escritorio\Universidad`).
2. Pega como primer mensaje el **PROMPT DE ARRANQUE** que está al final de este documento.
3. El hilo nuevo debe leer este archivo completo y seguir desde la sección "Próximo paso inmediato".

Este documento reemplaza al handoff anterior (`HANDOFF_torneo-wako_2026-08-10.md`) — ese ya se
completó (la migración a Firebase y la publicación en GitHub Pages, que eran su objetivo, están
hechas hace rato). Este es el handoff **actualizado** con todo lo que pasó después.

---

## Contexto del proyecto

App de torneo para la **Academia Taekwon-Do ITF Pukem Kwan** (Castro, Chiloé — instructor Marcelo
Vergara). Hoy gestiona un torneo de **kickboxing WAKO** (Point Fighting, Light Contact, Kick Light).
Es una sola página HTML autocontenida (sin build step, JS vanilla, sin frameworks, todo — incluida
la tipografía y el logo — embebido en el mismo archivo) pensada para usarse desde el celular durante
el torneo: inscripción de luchadores, generación de llaves, marcador en vivo sincronizado entre
varios dispositivos (jueces, jefe de mesa, pantalla pública), y ahora preparada para **varios
tatamis simultáneos**.

- **Archivo de trabajo (edita este):** `torneo-wako-pukemkwan_2.html` (esta misma carpeta).
- **Repo público / lo que se publica:** carpeta aparte `C:\Users\luzug\OneDrive\Escritorio\torneo-wako-pukemkwan\`
  (archivo `index.html` ahí adentro), repo de GitHub `Crisvalver9/torneo-wako-pukemkwan`.
- **URL pública (GitHub Pages):** https://crisvalver9.github.io/torneo-wako-pukemkwan/
- **Base de datos:** Firebase Realtime Database — `https://sistema-pukem-kwan-default-rtdb.firebaseio.com`
  (reglas abiertas `{read:true, write:true}`, decisión aceptada para un torneo interno de bajo riesgo).

## Qué se hizo en sesiones anteriores (resumen, ya terminado)

- Migración completa de `window.storage` (API exclusiva de artifacts de Claude.ai) a Firebase
  Realtime Database vía `fetch()`.
- Repo de GitHub creado, GitHub Pages activado, primera publicación exitosa.
- Point Fighting rediseñado según el reglamento real: los jueces señalan en el tatami, el **Jefe
  de Mesa** carga los puntos (ya no hay voto por celular en PF). Light Contact / Kick Light: cada
  juez sigue anotando desde su propio celular, independiente.
- Cierre automático de combate según el reglamento WAKO (Art. 6.2 / 6.4): se acaba solo al terminar
  el tiempo del último asalto; empate en PF → 1 min extra + muerte súbita; empate en LC/KL → aviso
  para que la mesa decida por criterio (no se puede automatizar, lo dice el reglamento).
- Pantalla de "Gana X" (nombre, club, marcador) sincronizada entre Mesa y Pantalla Pública al
  terminar un combate.
- Árbol de llave con líneas conectoras en Pantalla Pública (Octavos→Cuartos→Semis→Final); 3er/4to
  puesto aparte porque no es parte de esa cadena lineal.
- Logo oficial de la academia embebido (favicon + pantalla de inicio + pantalla pública).
- Tipografía propia (Barlow Condensed, embebida) en vez de la fuente genérica del sistema.
- Ortografía/tildes corregidas en todo el texto visible.
- Colores unificados: rojo = esquina Rojo siempre; ámbar = "atención/estado" (antes "muerte
  súbita"/"empate total" usaban rojo, generaba confusión).
- Micro-interacciones (los botones responden al tocarlos).

## Qué se hizo en la sesión más reciente (la que generó este handoff)

1. **Bug crítico de arquitectura corregido**: antes, cada acción (un punto, una falta, etc.)
   reescribía el documento **completo** de Firebase (`/torneo.json`). Con varios tatamis/jueces
   usando la app a la vez, la última escritura ganaba y borraba en silencio cambios de otras
   categorías hechos segundos antes. Ahora cada categoría se guarda en su propio nodo
   (`/torneo/categories/{id}.json`) vía `putCategory()`/`deleteCategoryRemote()`. Además se agregó
   un contador `writeSeq` que descarta lecturas de sincronización que llegan tarde, para que no
   "revivan" un combate que el usuario ya cerró (esto era el bug de "se queda pegado" que se
   reportó).
2. **Soporte multi-tatami**: cada categoría tiene un campo `tatami` de **texto libre** (no un
   número fijo — el usuario fue explícito: pueden ser 1, 2, 3, 8, 10 tatamis, el sistema no debe
   asumir una cantidad). Las listas de Mesa, Jefe de Mesa, Juez y Pantalla Pública agrupan los
   combates por tatami (`groupByTatami`/`renderTatamiGroups`).
3. **PIN opcional** (`configurarPin()`, `checkPin()`) protege solo lo delicado: declarar ganador
   manual, reabrir llave, eliminar categoría, y **todo** el rol de Jefe de Mesa. Inscribir
   luchadores, crear/cerrar llaves y votar como juez quedan libres para cualquiera — así lo pidió
   el usuario explícitamente, para evitar que alguien altere resultados del campeonato sin
   estorbar la operación normal.
4. **Respaldo descargable** (`descargarRespaldo()`): botón en Mesa que baja un JSON con todas las
   categorías/resultados, por si hay que limpiar Firebase para el próximo torneo.
5. **Jefe de Mesa rediseñado como consola compacta**: todo el control (tiempo, marcador, faltas,
   cierre anticipado) cabe en una pantalla de celular sin scroll — antes requería bastante scroll.
6. **Menos fricción**: el formulario de nueva categoría ahora es "pegajoso" — solo se resetean
   nombre/división/peso al crear una categoría; modalidad/sexo/cinturón/jueces/asaltos/duración/
   tatami quedan como la mesa los dejó la última vez (la mesa suele crear varias categorías
   seguidas del mismo formato).
7. **Sorteo de llave corregido (importante)**: antes los inscritos se colocaban en las casillas en
   orden secuencial de inscripción. Con menos de 16 luchadores, esto dejaba **toda la mitad
   inferior de la llave vacía** — una semifinal completa sin combate real, Final casi automática.
   Ahora se mezclan al azar y se distribuyen con el orden de siembra estándar de un cuadro de 16
   (`BRACKET_SEED_ORDER = [0,15,7,8,3,12,4,11,1,14,6,9,2,13,5,10]`, calculado y verificado con
   Python), que reparte personas y "bye" parejo por toda la llave sin importar cuántos inscritos
   haya.
8. **Verificación de pesaje** (`togglePesado()`): botón "pesar"/"✓ pesado" por luchador en la lista
   de inscritos, aviso (no bloqueante) si falta alguien por marcar al cerrar inscripciones.
9. **Certificado de resultados imprimible** (`renderCertificado()`, vista `mesa_certificate`):
   podio 1°–4° con nombre y club, botón "Imprimir/PDF" (`window.print()`), CSS `@media print`
   dedicado (fondo blanco, texto negro, oculta la barra de navegación).
10. **Bug crítico encontrado y corregido (el más reciente)**: Firebase Realtime Database es
    inconsistente devolviendo arreglos dispersos. El arreglo `slots` de la llave (16 casillas, la
    mayoría `null` cuando hay menos de 16 inscritos) a veces volvía desde Firebase como **objeto
    disperso** (`{"0":.., "15":..}`) en vez de arreglo real, y `buildBracket()` no lo reconocía —
    trataba toda la llave como vacía. Esto rompía el podio y el certificado en cualquier torneo
    con menos de 16 luchadores (o sea, casi siempre). Se agregó `arrayFromMaybeSparse(v, length)`
    en la capa de normalización (`normalizeCategory`), que siempre reconstruye un arreglo de 16
    posiciones sin importar cómo haya vuelto desde Firebase. Verificado con 2 y 9 luchadores
    pasando por una escritura y lectura real a Firebase, hasta calcular el podio final bien.
11. Se instaló **GitHub CLI** (`gh`, en `C:\Program Files\GitHub CLI\gh.exe`) pero **no está
    autenticado todavía** — falta correr `gh auth login` (requiere interacción del usuario, no se
    puede hacer solo).

## Decisiones ya tomadas (no volver a preguntar)

- Arquitectura: un solo archivo HTML, JS vanilla, sin framework, sin build step. Fuente y logo
  embebidos como base64 para mantenerlo autocontenido. Mantener así.
- Hosting: **GitHub Pages**, no Vercel (se evaluó explícitamente y se descartó Vercel — no aporta
  nada para un archivo estático sin build ni backend propio).
- El usuario **activa/desactiva GitHub Pages él mismo** cuando corresponde — no hace falta
  recordárselo ni ofrecerse a hacerlo.
- Firebase con reglas abiertas: riesgo aceptado para torneo interno. La URL de Firebase **no es
  secreta** (se ve en el código fuente público), eso ya se le explicó al usuario y lo entiende.
- PIN: protege únicamente declarar ganador manual, reabrir llave, eliminar categoría, y el rol
  Jefe de Mesa completo. Todo lo demás queda libre. No expandir la protección sin que el usuario
  lo pida — el criterio es "evitar alterar resultados", no "bloquear todo".
- Tatami: campo de texto libre, nunca un número fijo/dropdown limitado.
- **Nunca tocar la lógica de progresión de la llave** (`leafMatch`/`nextRoundMatch`/`buildBracket`)
  sin volver a probar exhaustivamente los casos de bye/waiting/empty — ya mordió dos veces en este
  proyecto (una vez advertido en el handoff original, y el bug de `slots` disperso recién
  corregido). Cualquier cambio ahí necesita pruebas con 2, 3, 5, 9, 16 luchadores como mínimo,
  pasando por una escritura/lectura real a Firebase (no solo estado local en memoria).
- Este computador **no tiene identidad de git configurada globalmente** — nunca correr
  `git config --global`. Cada commit se hace con `git -c user.name="Cristobal Valverde" -c
  user.email="valverdecristobal91@gmail.com" commit -m "..."`.
- Confirmar con el usuario antes de hacer `git push`, **excepto** cuando el cambio es chico,
  seguro y ya está probado (CSS, ajustes menores) — el usuario pidió explícitamente reducir la
  fricción de estar preguntando por cosas rutinarias. Ante la duda, preguntar.
- Idioma: español de Chile, tono directo y práctico. El usuario **no es programador** (es
  estudiante de Ingeniería Civil Industrial) — explicar en simple, sin asumir jerga.

## Estado AHORA MISMO

- Todo lo descrito arriba está **subido y publicado** en la URL pública.
- No hay ningún cambio sin subir pendiente al momento de escribir este handoff.
- `gh` está instalado pero sin login — si hace falta usarlo (por ejemplo, para prender/apagar
  GitHub Pages por API en vez de manual), primero hay que correr `gh auth login` con el usuario
  presente (es interactivo).
- Servidor local de pruebas: se suele levantar con `python -m http.server 8765` en esta carpeta
  para probar antes de subir. **Ojo**: ese servidor "local" sigue hablando con el Firebase real de
  producción — no hay una base de datos separada para pruebas. Siempre limpiar después de probar:
  `curl -s -X DELETE "https://sistema-pukem-kwan-default-rtdb.firebaseio.com/torneo.json"`.

### Trampas ya conocidas (para no perder tiempo redescubriéndolas)

- **Caché del navegador**: después de editar el HTML, `navigate` con `force:true` a veces sigue
  sirviendo una versión vieja cacheada. Si algo "no cambió" después de una edición, agregar un
  query string (`?v=2`, `?v=3`, ...) a la URL para forzar la recarga real antes de sospechar de un
  bug en el código.
- **Carreras en scripts de prueba**: llamar varias funciones de la app seguidas sin esperar (por
  ejemplo `createCategory()` inmediatamente después de `goTo()`, que dispara un `loadTorneo()`
  async) puede pisar el estado local con una lectura vieja de Firebase y parecer un bug que en
  realidad es solo el script de prueba mal armado. Siempre `clearTimers()` + esperar ~800ms antes
  de manipular `state` directamente en pruebas, y volver a leer el estado fresco después de cada
  escritura en vez de confiar en variables ya capturadas.
- **El Grep tiene un bug cosmético de visualización**: a veces muestra `</div>` como `<\div>`, o
  `·` como caracteres raros, en su propio output — pero el archivo real está bien. Antes de
  reportar algo como bug de código, confirmar con `Read` (no solo con `Grep`).

## Próximo paso inmediato

No hay una tarea a medio terminar — el próximo paso es **preguntarle al usuario qué sigue**. Lo
que quedó pendiente/mencionado y no se ha hecho:

1. **ITF Taekwondo** (visión de más largo plazo del usuario — la academia es de Taekwon-Do ITF,
   WAKO kickboxing es el torneo actual). Ya se acordó explícitamente: primero terminar de pulir
   WAKO, y cuando se retome, empezar por **ITF Sparring** (semi-contacto, sistema de puntos con
   parada, 4 jueces de esquina — estructura parecida a lo que ya existe) y dejar **Patrones/Tul,
   Técnica Especial y Rotura de Tablas para después**, porque usan modelos de juzgamiento
   totalmente distintos (comparación cara a cara con banderas/notas numéricas, no combate punto a
   punto) y necesitarían pantallas nuevas, no una extensión de lo que ya hay.
2. Seguir profundizando en Sportdata si el usuario lo pide (ya se investigó bastante: ecosistema
   de apps, rol del juez central, manejo de multi-tatami, pesaje, certificados — puede que quiera
   ir más allá).
3. Revisión de colores/estética más fina si el usuario encuentra algo al usar la app.
4. Autenticar `gh` si se necesita automatizar algo de GitHub que hoy se hace a mano.

Al retomar: pregunta directamente "¿en qué seguimos?" en vez de asumir. El usuario ha estado dando
mucha instrucción de una vez en mensajes largos (a veces por dictado de voz, con errores de
tipeo/gramática) — conviene parsear con calma, confirmar antes de cambios grandes de arquitectura
(como se hizo con multi-tatami y PIN), y proceder directo en cambios chicos ya confirmados como
seguros.

## Reglas de trabajo con el usuario

- Habla en español (Chile), tono directo y práctico — no es programador, explica en simple.
- Antes de tocar la lógica de la llave/bracket: releer la advertencia arriba y probar a fondo.
- Antes de `git push`: confirmar, salvo cambios chicos/seguros ya probados.
- El usuario activa/desactiva GitHub Pages solo — no ofrecerse a hacerlo ni recordárselo.
- No inventar URLs, claves ni credenciales.
- Cuando el usuario mande un mensaje muy largo con varios pedidos mezclados: separarlos, priorizar
  lo concreto y de bajo riesgo primero, y preguntar (con `AskUserQuestion`) antes de cambios de
  arquitectura o de alcance grande (como se hizo con el número de tatamis y el nivel de PIN).
- Limpiar siempre los datos de prueba de Firebase después de probar (`curl -X DELETE ...`).

## Archivos clave

- `torneo-wako-pukemkwan_2.html` (esta carpeta) — archivo de trabajo, edítalo aquí.
- `..\torneo-wako-pukemkwan\index.html` — copia que se publica (cp + git commit + push desde ahí).
- `HANDOFF_torneo-wako_2026-08-10.md` (esta carpeta) — handoff anterior, ya completado, solo como
  referencia histórica.
- Este archivo.

## PROMPT DE ARRANQUE

```
Lee el archivo HANDOFF_torneo-wako_2026-08-14.md en esta carpeta completo y sigue desde la sección
"Próximo paso inmediato". Es la app de torneo WAKO de la Academia Pukem Kwan (un solo HTML,
Firebase, publicada en GitHub Pages) — ya tiene mucho trabajo hecho, no es un proyecto nuevo.
No soy programador, así que ve explicándome cada paso en simple.
```
