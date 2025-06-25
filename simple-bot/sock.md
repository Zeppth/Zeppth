
# Objeto `sock` y sus Helpers

El objeto `sock` es la instancia principal de la conexión con WhatsApp, potenciada por Baileys. En este bot, ha sido enriquecido con una serie de funciones de ayuda (`helpers`) personalizadas que simplifican enormemente las tareas complejas y repetitivas. Dominar `sock` te permitirá llevar tus plugins al siguiente nivel.

## Índice
1.  [Obtención de Datos (Entidades)](#1-obtención-de-datos-entidades)
    -   [`sock['user:data']`](#sockuserdatauserjid)
    -   [`sock['chat:data']`](#sockchatdatachatjid)
2.  [Envío de Contenido Avanzado](#2-envío-de-contenido-avanzado)
    -   [`sock.sendButton`](#socksendbuttonjid-object-options)
    -   [`sock.sendWAMContent`](#socksendwamcontentjid-message-options)
3.  [Manejo de Archivos y Multimedia](#3-manejo-de-archivos-y-multimedia)
    -   [`sock.getFrom`](#sockgetfromsource-type)
    -   [`sock.downloadMedia`](#sockdownloadmediamessage-type)
    -   [`sock.resizePhoto`](#sockresizephotodata)
    -   [`sock.uploadFiloTmp`](#sockuploadfilotmpfile)
4.  [Utilidades de Red](#4-utilidades-de-red)
    -   [`sock.getJSON`](#sockgetjsonurl)
5.  [Sistema Interactivo](#5-sistema-interactivo)
    -   [`sock.saveMessageIdForResponse`](#socksavemessageidforresponse)

---

## 1. Obtención de Datos (Entidades)

Estas funciones son abstracciones para obtener información detallada sobre usuarios y chats de una manera sencilla y cacheada.

### `sock['user:data'](userJid)`
Obtiene un objeto enriquecido con los datos de un usuario específico.

-   **Retorna**: Un objeto con las siguientes propiedades:
    -   `id`: El JID completo del usuario.
    -   `number`: El número de teléfono.
    -   `bot`: Booleano que indica si el usuario es el bot.
    -   `photo(type)`: Función asíncrona que devuelve la URL de la foto de perfil ('image' o 'preview').
    -   `desc()/`: Función asíncrona que devuelve el estado/info del usuario.
    -   `waLink`: El enlace directo `wa.me`.

**Ejemplo de uso (comando `!profile`):**
```javascript
// /plugins/profile.js
export default {
    command: true, usePrefix: true, case: 'profile',
    script: async (m, { sock }) => {
        const target = m.message.extendedTextMessage?.contextInfo?.mentionedJid?.[0] || m.quoted?.sender.id || m.sender.id;
        const userData = await sock['user:data'](target);
        
        const profileMessage = `
*Perfil de @${userData.number}*

*Nombre:* ${userData.name || 'Sin nombre'}
*Info:* ${await userData.desc()}
*Enlace:* ${userData.waLink}
        `.trim();

        await sock.sendMessage(m.chat.id, {
            image: { url: await userData.photo('image') },
            caption: profileMessage,
            mentions: [target]
        });
    }
};
```

### `sock['chat:data'](chatJid)`
Obtiene un objeto enriquecido con los metadatos y métodos de un grupo.

-   **Retorna**: Un objeto con propiedades como `id`, `name`, `desc`, `participants`, `admins`, `owner`, y métodos de moderación como `promote()`, `demote()`, `change.name()`, etc., si el bot es admin. (Ver documentación de `m.chat` para una lista completa, ya que esta función es la base).

---

## 2. Envío de Contenido Avanzado

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
                    display_text: 'Ver comandos',
                    id: '!comandos' // El texto que el bot recibirá cuando se presione
                })
            },
            {
                name: 'cta_url',
                buttonParamsJson: JSON.stringify({
                    display_text: 'Visitar web',
                    url: 'https://github.com'
                })
            }
        ];

        await sock.sendButton(m.chat.id, {
            image: { url: 'https://i.imgur.com/......jpg' },
            body: 'Bienvenido al menú principal. ¿Qué deseas hacer?',
            footer: 'Bot de Zeppth',
            buttons: menuButtons
        });
    }
};
```

### `sock.sendWAMContent(jid, message, options)`
Es una función de más bajo nivel para enviar un objeto de mensaje de Baileys ya construido. `m.reply` y `sock.sendButton` la usan internamente. Es útil cuando necesitas control total sobre el objeto del mensaje.

---

## 3. Manejo de Archivos y Multimedia

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
Descarga multimedia de un objeto de mensaje de Baileys. Es un alias del `downloadMediaMessage` de Baileys.

### `sock.resizePhoto(data)`
Redimensiona una imagen, útil para fotos de perfil.

-   **`data`**: Un objeto con `{ image, scale, result }`.
    -   `image`: Buffer o URL de la imagen.
    -   `scale`: Tamaño en píxeles al que se ajustará (default 720).
    -   `result`: Formato de salida, `'buffer'` o `'base64'`.

### `sock.uploadFiloTmp(file)`
Sube un archivo (Buffer o URL) a un servicio de alojamiento temporal y devuelve el enlace de descarga directa.

---

## 4. Utilidades de Red

### `sock.getJSON(url)`
Realiza una petición GET a una URL y parsea la respuesta como JSON automáticamente.

**Ejemplo de uso (comando de meme):**
```javascript
// /plugins/meme.js
export default {
    command: true, usePrefix: true, case: 'meme',
    script: async (m, { sock }) => {
        try {
            const res = await sock.getJSON('https://meme-api.com/gimme');
            await sock.sendMessage(m.chat.id, {
                image: { url: res.url },
                caption: `${res.title}\n_r/${res.subreddit}_`
            });
        } catch (e) {
            m.reply('No se pudo obtener un meme en este momento.');
        }
    }
};
```

## 5. Sistema Interactivo

### `sock.saveMessageIdForResponse`
Permite crear flujos de conversación. Guarda una "promesa" de que la próxima respuesta de un usuario a un mensaje específico del bot debe ser tratada de una manera especial. Se usa en conjunto con la lógica de respuesta en `script.js`.

Es un concepto avanzado. Su uso principal es para construir comandos que hacen preguntas y esperan una respuesta específica.

**Ejemplo conceptual:**
1.  El bot envía un mensaje: "¿Cuál es tu nombre?" y guarda el ID de ese mensaje con `sock.saveMessageIdForResponse`.
2.  El usuario responde a ESE mensaje con "Juan".
3.  El sistema en `script.js` detecta que es una respuesta a un mensaje guardado, busca la configuración, y procesa "Juan" como la respuesta a la pregunta del nombre.
