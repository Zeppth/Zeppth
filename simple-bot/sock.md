
# Objeto `sock` y sus Helpers

El objeto `sock` es la instancia principal de la conexión con WhatsApp, potenciada por Baileys. Mientras que el objeto `m` te da todo el contexto del mensaje *actual*, `sock` te da el poder de actuar de forma independiente: enviar mensajes a cualquier chat, obtener información, o usar funciones avanzadas que no dependen de un mensaje entrante.

En este bot, `sock` ha sido enriquecido con funciones de ayuda (`helpers`) que simplifican tareas complejas.

## Índice
1.  [Envío de Contenido Avanzado](#1-envío-de-contenido-avanzado)
    -   [`sock.sendButton`](#socksendbuttonjid-object-options)
    -   [`sock.sendWAMContent`](#socksendwamcontentjid-message-options)
2.  [Manejo de Archivos y Multimedia](#2-manejo-de-archivos-y-multimedia)
    -   [`sock.getFrom`](#sockgetfromsource-type)
    -   [`sock.downloadMedia`](#sockdownloadmediamessage-type)
    -   [`sock.resizePhoto`](#sockresizephotodata)
    -   [`sock.uploadFiloTmp`](#sockuploadfilotmpfile)
3.  [Utilidades de Red](#3-utilidades-de-red)
    -   [`sock.getJSON`](#sockgetjsonurl)
4.  [Sistema Interactivo: `sock.saveMessageIdForResponse`](#4-sistema-interactivo-socksavemessageidforresponse)
    -   [El Problema: La "Amnesia" de los Comandos](#el-problema-la-amnesia-de-los-comandos)
    -   [La Solución y su Anatomía](#la-solución-y-su-anatomía)
    -   [Ejemplos Prácticos de `saveMessageIdForResponse`](#ejemplos-prácticos-de-savemessageidforresponse)
5.  [Caso de Uso: ¿Cuándo usar `sock` en lugar de `m`?](#5-caso-de-uso-cuándo-usar-sock-en-lugar-de-m)

---

## 1. Envío de Contenido Avanzado

### `sock.sendButton(jid, object, options)`
Envía un mensaje interactivo complejo con cabecera, cuerpo, pie de página y botones. También puede incluir una imagen, video o documento en la cabecera.

-   **`jid`**: El JID del chat al que se enviará.
-   **`object`**: Un objeto con la configuración del mensaje:
    -   `title`: Texto en la cabecera (si no hay multimedia).
    -   `body`: Texto principal del mensaje.
    -   `footer`: Texto en el pie de página.
    -   `buttons`: Un array de objetos de botón.
    -   `image | video | document`: Un Buffer o URL para mostrar en la cabecera.

**Ejemplo de uso (menú con botones):**
```javascript
// /plugins/button-menu.js
export default {
    command: true, usePrefix: true, case: 'menu',
    script: async (m, { sock }) => {
        const menuButtons = [
            {
                name: 'quick_reply',
                buttonParamsJson: JSON.stringify({
                    display_text: 'Ver Comandos de Diversión',
                    id: '!comandos diversion'
                })
            },
            {
                name: 'cta_url',
                buttonParamsJson: JSON.stringify({
                    display_text: 'Visitar mi Repositorio',
                    url: 'https://github.com/Zeppth'
                })
            }
        ];

        await sock.sendButton(m.chat.id, {
            image: { url: 'https://i.imgur.com/example.jpg' },
            body: 'Bienvenido al menú principal. ¿Qué deseas hacer?',
            footer: 'Bot Creado por Zeppth',
            buttons: menuButtons
        });
    }
};
```

### `sock.sendWAMContent(jid, message, options)`
Es una función de más bajo nivel para enviar un objeto de mensaje de Baileys ya construido. `m.reply` y `sock.sendButton` la usan internamente. Es útil cuando necesitas control total sobre el objeto del mensaje.

---

## 2. Manejo de Archivos y Multimedia

### `sock.getFrom(source, [type])`
Una utilidad muy versátil para obtener un Buffer a partir de diversas fuentes.

-   **`source`**: Puede ser una URL, una ruta de archivo local, un Buffer, o un Stream.
-   **`type`**: Formato de salida. Puede ser `'buffer'` (default), `'stream'`, o `'base64'`.

**Ejemplo de uso (sticker desde URL):**
```javascript
// /plugins/sticker-from-url.js
export default {
    command: true, usePrefix: true, case: 'surl',
    script: async (m, { sock }) => {
        if (!m.text || !m.text.startsWith('http')) {
            return m.reply('Por favor, proporciona una URL de imagen válida.');
        }
        try {
            const imageBuffer = await sock.getFrom(m.text, 'buffer');
            await sock.sendMessage(m.chat.id, { sticker: imageBuffer }, { quoted: m.message });
        } catch (e) {
            m.reply('No se pudo crear el sticker desde esa URL.');
        }
    }
};
```

### `sock.downloadMedia(message, [type])`
Descarga multimedia de un objeto de mensaje de Baileys. Es un alias del `downloadMediaMessage` de Baileys y la base para `m.download()` y `m.quoted.download()`.

### `sock.resizePhoto(data)`
Redimensiona una imagen, útil para fotos de perfil.

-   **`data`**: Un objeto con `{ image, scale, result }`.
    -   `image`: Buffer o URL de la imagen.
    -   `scale`: Tamaño en píxeles al que se ajustará (default 720).
    -   `result`: Formato de salida, `'buffer'` o `'base64'`.

### `sock.uploadFiloTmp(file)`
Sube un archivo (Buffer o URL) a un servicio de alojamiento temporal y devuelve el enlace de descarga directa.

---

## 3. Utilidades de Red

### `sock.getJSON(url)`
Realiza una petición GET a una URL y parsea la respuesta como JSON automáticamente.

**Ejemplo de uso (comando de frase célebre):**
```javascript
// /plugins/quote.js
export default {
    command: true, usePrefix: true, case: 'frase',
    script: async (m, { sock }) => {
        try {
            const res = await sock.getJSON('https://api.quotable.io/random');
            await m.reply(`"*${res.content}*"\n\n- ${res.author}`);
        } catch (e) {
            m.reply('No se pudo obtener una frase en este momento.');
        }
    }
};
```

---

## 4. Sistema Interactivo: `sock.saveMessageIdForResponse`

### El Problema: La "Amnesia" de los Comandos
Por naturaleza, cada comando es independiente. Si un bot pregunta "¿Estás seguro?", no tiene forma de saber que el siguiente mensaje del usuario ("sí") está relacionado con esa pregunta.

### La Solución y su Anatomía
La función `sock.saveMessageIdForResponse(message, config)` resuelve esto creando un "oyente" temporal en el mensaje que el bot envía.

-   **`message`** `(Object)`: El objeto del mensaje que **TÚ envías** y que esperas que el usuario responda. Lo obtienes como retorno de `sock.sendMessage`.
-   **`config`** `(Object)`: Un objeto que define las reglas:
    -   **`user`** `(String)`: Quién puede responder (`'all'` o `m.sender.id`).
    -   **`response`** `(Array<Object>)`: Un array de reglas. Cada regla tiene:
        -   **`condition(m)`**: Una función que debe devolver `true` si la respuesta del usuario es válida para esta regla.
        -   **`command`**: El nuevo comando a ejecutar si la condición es `true`.
        -   **`dynamic` y `extract`**: Para casos avanzados donde el nuevo comando se construye con el texto del usuario.

### Ejemplos Prácticos de `saveMessageIdForResponse`

#### Ejemplo 1: Confirmación Simple (Sí/No)
```javascript
// /plugins/delete-data.js
export default {
    command: true, usePrefix: true, case: 'borrardatos',
    script: async (m, { sock }) => {
        const confirmationMsg = await sock.sendMessage(m.chat.id, {
            text: '⚠️ ¿Estás SEGURO de que quieres borrar todos tus datos? Responde `SÍ, BORRAR` para confirmar.'
        }, { quoted: m });

        await sock.saveMessageIdForResponse(confirmationMsg, {
            user: m.sender.id, // Solo el usuario original puede confirmar
            response: [
                {
                    condition: (msg) => msg.body === 'SÍ, BORRAR',
                    command: '.confirmar-borrado' // Ejecuta otro comando que hace la lógica
                }
            ]
        });
    }
};
```

#### Ejemplo 2: Menú Numérico Dinámico
```javascript
// /plugins/search-wallpaper.js
export default {
    command: true, usePrefix: true, case: 'buscarfondo',
    script: async (m, { sock }) => {
        if (!m.text) return m.reply('¿Qué tipo de fondo de pantalla buscas?');
        
        // Simulación de búsqueda
        const results = [
            { title: 'Bosque Nebuloso', url: 'https://example.com/img1.jpg' },
            { title: 'Ciudad Nocturna', url: 'https://example.com/img2.jpg' },
            { title: 'Playa Tropical', url: 'https://example.com/img3.jpg' }
        ];

        let responseText = 'Resultados de búsqueda:\n\n';
        // Escapamos los caracteres especiales para la constante de JS
        responseText += results.map((item, i) => `*${i + 1}.* ${item.title}`).join('\n');
        responseText += '\n\nResponde con el número de la imagen que quieres.';

        const responseRules = results.map((item, i) => ({
            condition: (msg) => msg.body.trim() === `${i + 1}`,
            command: `.enviarfondo ${item.url}`
        }));
        
        const searchMsg = await m.reply(responseText);
        await sock.saveMessageIdForResponse(searchMsg, {
            user: m.sender.id,
            response: responseRules
        });
    }
};

// Plugin que maneja la respuesta
/*
export default {
    command: true, usePrefix: true, case: 'enviarfondo',
    script: async (m, { sock }) => {
        // m.text aquí contendrá la URL
        await sock.sendMessage(m.chat.id, { image: { url: m.text } });
    }
}
*/
```
---

## 5. Caso de Uso: ¿Cuándo usar `sock` en lugar de `m`?

La regla general es:
-   **Usa `m`** para todo lo que esté relacionado con el mensaje actual: responder (`m.reply`), reaccionar (`m.react`), obtener datos del remitente (`m.sender`) o del chat actual (`m.chat`).
-   **Usa `sock`** cuando necesites hacer algo que vaya más allá del contexto inmediato.

**Ejemplos claros para usar `sock`:**
-   Enviar un mensaje a un chat diferente al actual.
-   Crear un sticker a partir de una URL externa.
-   Enviar un mensaje de bienvenida en un plugin `stubtype`.
-   Modificar la foto de perfil del propio bot.
-   Enviar mensajes con componentes complejos como botones o plantillas.
