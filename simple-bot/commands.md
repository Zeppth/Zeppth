
# Sistema de Comandos

Los plugins de tipo `command` son el núcleo interactivo de tu bot. Permiten a los usuarios ejecutar acciones específicas enviando mensajes que coinciden con un patrón predefinido (generalmente un prefijo seguido de una palabra clave).

## Índice
1.  [¿Qué es un Plugin de Comando?](#1-qué-es-un-plugin-de-comando)
2.  [Estructura de un Plugin de Comando](#2-estructura-de-un-plugin-de-comando)
    -   [Propiedades Clave: `command`, `case` y `usePrefix`](#propiedades-clave-command-case-y-useprefix)
3.  [Manejo de Argumentos y Texto](#3-manejo-de-argumentos-y-texto)
4.  [Trabajando con Menciones y Mensajes Citados](#4-trabajando-con-menciones-y-mensajes-citados)
5.  [Validaciones Comunes](#5-validaciones-comunes)
6.  [Ejemplos Prácticos](#6-ejemplos-prácticos)
    -   [Comando Simple con Alias](#ejemplo-1-comando-simple-con-alias)
    -   [Comando que Requiere Argumentos](#ejemplo-2-comando-que-requiere-argumentos)
    -   [Comando de Administración (Kick)](#ejemplo-3-comando-de-administración-kick)
    -   [Comando sin Prefijo](#ejemplo-4-comando-sin-prefijo)

---

## 1. ¿Qué es un Plugin de Comando?

Es un módulo de código diseñado para ejecutarse cuando un usuario envía un mensaje que empieza con un prefijo (ej: `!`, `.`) y una palabra clave (ej: `ping`, `menu`). Su objetivo es tomar ese comando, procesar cualquier dato adicional (argumentos, menciones) y realizar una acción como respuesta.

---

## 2. Estructura de un Plugin de Comando

Todo plugin de comando debe exportar un objeto con, como mínimo, tres propiedades esenciales:

```javascript
export default {
    // Propiedades que definen el comportamiento del comando
    command: true,
    usePrefix: true,
    case: ['play', 'yt', 'reproducir'],

    // La función que contiene la lógica del comando
    script: async (m, { sock }) => {
        if (!m.text) {
            return m.reply('Debes proporcionar el nombre de una canción o un enlace de YouTube.');
        }
        await m.reply(`Buscando "${m.text}" en YouTube...`);
        // ...lógica para buscar y descargar la canción...
    }
};
```

### Propiedades Clave: `command`, `case` y `usePrefix`

-   **`command: true`**
    -   **Obligatorio**. Esta bandera le dice al sistema que este plugin es un comando y debe ser tratado como tal.

-   **`case: 'string' | ['array', 'de', 'strings']`**
    -   **Obligatorio**. Define la(s) palabra(s) clave que activarán el comando. Es **insensible a mayúsculas**.
    -   Si usas un **string** (`case: 'ping'`), solo esa palabra activará el comando.
    -   Si usas un **array** (`case: ['menu', 'help', 'ayuda']`), cualquiera de las palabras en el array servirá como un alias y activará el mismo comando.

-   **`usePrefix: true | false`**
    -   **Obligator रीड**. Controla si el comando necesita un prefijo para ser activado. Los prefijos válidos se definen en tu archivo `settings.json` (`mainBotPrefix`).
    -   **`true`**: El usuario debe escribir `!ping`, `.menu`, etc.
    -   **`false`**: El comando se activa solo con la palabra clave, sin prefijo. Útil para respuestas automáticas o "comandos ocultos" (ej: un plugin con `case: 'hola'` que responde "Hola!").

---

## 3. Manejo de Argumentos y Texto

Dentro del `script`, el objeto `m` te proporciona todo lo que necesitas para procesar la entrada del usuario:

-   **`m.text`**: Contiene todo el string que viene *después* de la palabra del comando. Es `null` si no hay nada más.
    -   Para `!kick @usuario por spam`, `m.text` sería `'@usuario por spam'`.
-   **`m.args`**: Es un array que contiene cada parte de `m.text` separada por espacios.
    -   Para `!kick @usuario por spam`, `m.args` sería `['@usuario', 'por', 'spam']`.
-   **`m.body`**: El texto completo del mensaje, tal como lo escribió el usuario.
    -   Para `!kick @usuario por spam`, `m.body` sería `'!kick @usuario por spam'`.

---

## 4. Trabajando con Menciones y Mensajes Citados

Es muy común que los comandos necesiten actuar sobre otro usuario. El bot facilita esto:

-   **Para obtener un usuario mencionado**:
    ```javascript
    const mentionedJid = m.message.extendedTextMessage?.contextInfo?.mentionedJid?.[0];
    ```
-   **Para obtener el autor de un mensaje citado (reply)**:
    ```javascript
    const quotedUserJid = m.quoted?.sender.id;
    ```
-   **Combinando ambos**: La mejor práctica es dar prioridad a la mención y, si no existe, usar el mensaje citado.
    ```javascript
    const targetUser = m.message.extendedTextMessage?.contextInfo?.mentionedJid?.[0] || m.quoted?.sender.id;
    ```

---

## 5. Validaciones Comunes

Antes de ejecutar la lógica principal, es buena práctica validar el contexto. El objeto `m` y sus helpers `m.sms()` simplifican esto:

-   **Solo grupos**: `if (!m.chat.group) return m.sms('group');`
-   **Solo privado**: `if (m.chat.group) return m.sms('private');`
-   **Solo admin**: `if (!m.sender.admin) return m.sms('admin');`
-   **Bot debe ser admin**: `if (!m.bot.admin) return m.sms('botAdmin');`
-   **Solo owner**: `if (!m.sender.owner) return m.sms('owner');`
-   **Requiere argumentos**: `if (!m.text) return m.reply('Faltan argumentos...');`

---

## 6. Ejemplos Prácticos

### Ejemplo 1: Comando Simple con Alias
```javascript
// /plugins/ping.js
export default {
    command: true,
    usePrefix: true,
    case: ['ping', 'test', 'prueba'],
    script: async (m) => {
        const startTime = Date.now();
        await m.react('⌛');
        const endTime = Date.now();
        const latency = endTime - startTime;
        await m.reply(`Pong! 🏓\nLatencia: ${latency} ms`);
    }
};
```

### Ejemplo 2: Comando que Requiere Argumentos
```javascript
// /plugins/weather.js
export default {
    command: true,
    usePrefix: true,
    case: 'clima',
    script: async (m) => {
        if (!m.text) {
            return m.reply('Por favor, especifica una ciudad. Ejemplo: ' + `!clima Madrid`);
        }
        // Lógica para buscar el clima de m.text...
        await m.reply(`Consultando el clima para: ${m.text}`);
    }
};
```

### Ejemplo 3: Comando de Administración (Kick)
```javascript
// /plugins/kick.js
export default {
    command: true,
    usePrefix: true,
    case: 'kick',
    script: async (m) => {
        if (!m.chat.group) return m.sms('group');
        if (!m.sender.admin) return m.sms('admin');
        if (!m.bot.admin) return m.sms('botAdmin');

        const target = m.message.extendedTextMessage?.contextInfo?.mentionedJid?.[0] || m.quoted?.sender.id;
        if (!target) return m.reply('Debes mencionar a alguien o responder a su mensaje.');

        try {
            await m.chat.remove(target);
            await m.reply('✅ Usuario expulsado con éxito.');
        } catch (e) {
            console.error(e);
            await m.reply('❌ Ocurrió un error al intentar expulsar al usuario.');
        }
    }
};
```

### Ejemplo 4: Comando sin Prefijo
```javascript
// /plugins/autoreply-hello.js
export default {
    command: true,
    usePrefix: false, // No necesita '!' o '.'
    case: 'hola',
    script: async (m) => {
        // Para evitar spam, solo responde si el mensaje es exactamente 'hola'
        if (m.body.toLowerCase() !== 'hola') return;
        
        await m.reply(`¡Hola, ${m.sender.name}! ¿En qué puedo ayudarte?`);
    }
};
```
