---
name: youtube-summary
description: Genera un resumen narrativo breve de un vídeo de YouTube a partir de su enlace. Úsala SIEMPRE que el usuario pegue una URL de youtube.com o youtu.be y pida un resumen, que "le cuentes de qué va", una transcripción, o cualquier variante de "resume este vídeo" — incluso si no usa la palabra "resumen" explícitamente.
---

# Resumen de vídeos de YouTube

Esta skill convierte un enlace de YouTube en un resumen narrativo breve, escrito en el mismo idioma del vídeo original.

## Restricción importante del entorno

`web_fetch` no sirve para esto: solo lee el HTML estático de la primera carga, no ejecuta JavaScript ni hace clics, y el contenido de subtítulos se carga dinámicamente. Para conseguir la transcripción automáticamente hace falta un agente que controle un navegador real — la integración de **Claude en Chrome**.

El flujo tiene cuatro pasos, en este orden:

### Paso 1 — Herramienta de transcripción de terceros vía Claude en Chrome (camino principal)
Comprueba si tienes disponibles las herramientas `mcp__claude-in-chrome__*`. Si no aparecen en tu lista de herramientas cargadas, búscalas con `tool_search` (por ejemplo con la consulta "chrome browser navigate click"). Si tras buscar siguen sin estar disponibles (la integración no está conectada en esta sesión), pasa directamente al Paso 4.

Si están disponibles, usa una herramienta web gratuita que obtiene transcripciones de YouTube a partir de la URL (por ejemplo `https://tactiq.io/es/herramientas/transcripcion-de-youtube`, aunque puede usarse cualquier alternativa equivalente si esa falla o cambia):

1. Navega a la herramienta (`navigate`).
2. Si aparece un aviso de cookies, elige la opción que menos datos comparte ("Decline"/"Rechazar"), no "Accept".
3. Localiza el campo de texto para pegar la URL del vídeo (`find`) e introduce ahí la URL exacta que pegó el usuario (`form_input`).
4. Localiza el botón de envío ("Get Video Transcript" o equivalente) y haz clic **por coordenadas de pantalla** (`computer` con `coordinate`, tomando antes un `screenshot`), no por `ref` — los clics por `ref` sobre botones dinámicos de estas páginas no siempre disparan el evento real.
5. Espera unos segundos y extrae el resultado con `get_page_text`.
6. Comprueba que lo extraído es una transcripción real (texto con marcas de tiempo y frases con sentido), no la landing page de la herramienta sin resultado.

Este método es más fiable que interactuar directamente con la página de YouTube (ver Paso 2) porque no sufre el problema de detección de automatización descrito ahí abajo. Si la herramienta falla, da un resultado vacío, o está caída, no insistas más de un par de intentos (incluyendo probar una alternativa si conoces otra) y pasa al Paso 2.

### Paso 2 — Interactuar directamente con la página de YouTube (respaldo)
Si el Paso 1 no está disponible o falla:
1. Navega a la URL exacta que pegó el usuario (`navigate`).
2. Localiza el botón **"...más"** bajo el título y los datos del vídeo (el que expande la descripción completa). Usa `find` para localizarlo, pero haz clic **por coordenadas de pantalla**, no por `ref`, por la misma razón del punto anterior. No es el botón "⋯" (más acciones, junto a "Compartir"/"Guardar") — ese abre un menú distinto y no contiene la transcripción.
3. Con la descripción ya expandida, toma un `screenshot`, localiza el botón **"Mostrar transcripción"** ("Show transcript") y haz clic también por coordenadas.
4. Justo después del clic, comprueba cuanto antes si la petición de red se disparó y qué código de estado tuvo: `read_network_requests` con `urlPattern: "transcript"`. No esperes a que el panel termine de cargar visualmente antes de mirar esto.
   - Si no aparece ninguna petición tras 1-2 segundos, el clic no se registró: puedes reintentarlo una vez (con coordenadas frescas de un nuevo screenshot, por si el layout cambió).
   - Si aparece con `statusCode` distinto de 200 (por ejemplo 400), no esperes a que el panel salga del estado de carga — no lo hará. Este es un fallo conocido: probablemente detección de automatización por parte de YouTube, que invalida la petición de transcripción para navegadores controlados por herramientas aunque el resto de la página cargue con normalidad. Pasa directamente al Paso 3.
5. Si la petición devolvió 200, extrae el texto del panel con `get_page_text` (o `read_page` si `get_page_text` no captura bien el panel lateral).
6. Comprueba que lo extraído es realmente la transcripción hablada. Los timestamps no son un problema — ignóralos al escribir el resumen.

No insistas más de un par de intentos en total en este paso: pasa al Paso 3.

**Limitación conocida**: incluso con clic real por coordenadas, YouTube puede rechazar la petición de transcripción específicamente cuando la hace un navegador controlado por una herramienta (error 400 que la interfaz no muestra, quedándose cargando indefinidamente). No es un fallo de la skill que se arregle ajustando los clics — es una barrera del lado de YouTube. Dilo así de claro si el usuario pregunta, en vez de sugerir que es un problema de configuración. Por esta razón el Paso 1 es el camino principal y este paso es solo un respaldo.

### Paso 3 — Intentar `web_fetch` como último respaldo automático
Si los pasos 1 y 2 no están disponibles o fallaron, haz `web_fetch` sobre la URL exacta que el usuario pegó. Rara vez trae la transcripción, pero a veces trae la descripción del vídeo o los capítulos, que pueden servir como base parcial si el resultado es texto coherente y sustancial (no metadatos ni menús).

### Paso 4 — Pedir la transcripción al usuario (último recurso)
Si ningún paso anterior dio una transcripción utilizable, pide al usuario que la pegue directamente. Indícale cómo conseguirla en segundos:

> "No he podido extraer la transcripción automáticamente. ¿Puedes pegarla? Se saca en YouTube: debajo del vídeo, pulsa '...más' para expandir la descripción → 'Mostrar transcripción', y copia el texto que aparece a la derecha."

No inventes ni completes contenido del vídeo que no esté en el texto obtenido o que te haya dado el usuario.

## Generar el resumen

Una vez tengas la transcripción (por el paso 1 o por el paso 2):

- **Formato**: resumen narrativo breve — un párrafo fluido (o dos como máximo si el vídeo es largo o cubre varios temas claramente distintos), no una lista de viñetas.
- **Idioma — regla estricta**: escribe el resumen en el idioma en que está hablado el vídeo original, detectado a partir del texto de la transcripción. Esta regla tiene prioridad sobre el idioma en el que transcurre la conversación con el usuario: si el vídeo está en inglés, el resumen va en inglés aunque el usuario te esté hablando en español (y viceversa). No traduzcas al idioma de la conversación salvo que el usuario lo pida explícitamente. Antes de escribir el resumen, identifica en una frase interna qué idioma detectas en la transcripción y no te desvíes de él.
- **Contenido**: qué trata el vídeo, la idea o argumento central, y los puntos que le dan cuerpo — evita simplemente repetir frases sueltas de la transcripción; sintetiza.
- **Longitud**: proporcional al vídeo, pero por defecto mantenlo breve (unas 4-8 frases). Si el usuario pide más o menos detalle, ajústalo.

## Casos límite

- **Vídeo sin transcripción disponible** (por ejemplo, sin subtítulos en absoluto, ni siquiera dentro de YouTube): dilo claramente y ofrece alternativas — que el usuario transcriba manualmente un fragmento, o que pegue la descripción del vídeo si tiene suficiente detalle.
- **Claude en Chrome no está conectado en esta sesión**: no lo trates como un error que hay que explicar en detalle; simplemente pasa al Paso 3/4 sin fricción. Solo menciona brevemente que no pudiste automatizarlo si terminas pidiendo el pegado manual, para que el usuario entienda por qué.
- **Transcripción muy larga**: resúmela igualmente en el formato breve definido arriba; no reproduzcas la transcripción completa ni la mayoría de sus frases textuales.
- **Vídeo en varios idiomas o con code-switching**: usa el idioma predominante.
- **El usuario pega solo el link sin pedir nada explícito**: interpreta que quiere el resumen (es el caso de uso principal de esta skill).
