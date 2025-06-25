#################################################################
#                                                               #
#          TUTORIAL PARA LA CREACIÓN DE PLUGINS DEL BOT         #
#                                                               #
#################################################################

Bienvenido a la guía de desarrollo para tu bot. Este sistema de plugins te permite
añadir nuevas funcionalidades de forma modular y organizada. Cada archivo `.js`
dentro de la carpeta `/plugins` es un plugin independiente.

---

### ÍNDICE

1.  Estructura Básica de un Plugin
2.  El Objeto `m` (o `data`): El Corazón de tu Plugin
    -   Propiedades Principales
    -   `m.chat`: Información y Acciones del Chat
    -   `m.sender`: Información y Acciones del Remitente
    -   `m.bot`: Información y Acciones del Bot
    -   `m.quoted`: Información del Mensaje Citado
    -   Helpers: `m.reply`, `m.react`, `m.sms`
3.  Propiedades del Plugin (El "Encabezado")
    -   `command`: Para comandos de texto.
    -   `stubtype`: Para eventos de grupo.
    -   `before`: Para ejecutar código antes de los comandos (middleware).
    -   `export`: Para compartir funciones entre plugins.
4.  Ejemplos Completos
    -   Comando simple: !ping
    -   Comando con argumentos: !say
    -   Comando de administrador: !kick
    -   Evento de bienvenida (stubtype)
    -   Middleware de registro (before)
    -   Sistema de export/import

---

### 1. ESTRUCTURA BÁSICA DE UN PLUGIN

Todos los plugins deben exportar un objeto por defecto. La propiedad más importante es `script`, que es la función que se ejecutará.

```javascript
// /plugins/mi-plugin.js

export default {
    // Aquí van las propiedades del plugin (ver sección 3)
    command: true,
    usePrefix: true,
    case: ['ping', 'test'],

    // Esta es la función principal que se ejecuta
    script: async (m, { sock, plugin }) => {
        // Tu código va aquí
        await m.reply('Pong!');
    }
};
```
-   `m`: El objeto principal con toda la información del mensaje (ver sección 2).
-   `sock`: El objeto de la conexión de Baileys, con todas las funciones de bajo nivel.
-   `plugin`: El manejador de plugins, útil para importar desde otros plugins.

---

### 2. EL OBJETO `m` (o `data`): EL CORAZÓN DE TU PLUGIN

El primer argumento de tu `script`, llamado `m` (o `data`), contiene todo lo que necesitas saber sobre el mensaje entrante.

#### Propiedades Principales:

-   `m.body`: (String) El contenido de texto del mensaje.
-   `m.command`: (String) El comando extraído del `body` (ej: 'ping').
-   `m.text`: (String) El resto del mensaje después del comando.
-   `m.args`: (Array<String>) `m.text` dividido por espacios.
-   `m.isCmd`: (Boolean) `true` si el mensaje fue identificado como un comando.
-   `m.plugin`: (Object) El objeto del plugin que se está ejecutando.
-   `m.message`: (Object) El objeto de mensaje crudo de Baileys.
-   `m.download()`: (Function) Descarga el adjunto del mensaje. `await m.download()` devuelve un Buffer.

#### `m.chat`: Información y Acciones del Chat

Contiene información sobre el chat donde se envió el mensaje.

-   `m.chat.id`: (String) El JID del chat (ej: '12345@c.us' o '12345-6789@g.us').
-   `m.chat.group`: (Boolean) `true` si el chat es un grupo.

**Propiedades y métodos solo para grupos (`if (m.chat.group)`):**
-   `m.chat.name`: (String) El nombre del grupo.
-   `m.chat.desc`: (String) La descripción del grupo.
-   `m.chat.participants`: (Array) Lista de participantes.
-   `m.chat.admins`: (Array<String>) Lista de JIDs de los administradores.
-   `m.chat.owner`: (String) El JID del dueño del grupo.
-   `m.chat.promote(userJid)`: Promueve a un usuario a administrador.
-   `m.chat.demote(userJid)`: Degrada a un administrador a miembro.
-   `m.chat.remove(userJid)`: Expulsa a un usuario.
-   `m.chat.photo('image' | 'preview')`: Obtiene la URL de la foto del grupo.
-   `m.chat.settings.lock(true|false)`: Cierra o abre el grupo.
-   `m.chat.settings.announce(true|false)`: Activa o desactiva el modo "solo admins".
-   `m.chat.update.name(new_name)`: Cambia el nombre del grupo.
-   `m.chat.update.desc(new_desc)`: Cambia la descripción del grupo.

#### `m.sender`: Información y Acciones del Remitente

-   `m.sender.id`: (String) El JID de quien envió el mensaje.
-   `m.sender.name`: (String) El nombre de perfil (pushName).
-   `m.sender.number`: (String) El número de teléfono sin el `@s.whatsapp.net`.
-   `m.sender.admin`: (Boolean) `true` si el remitente es admin del grupo (solo en grupos).
-   `m.sender.rowner`: (Boolean) `true` si es el dueño principal del bot.
-   `m.sender.owner`: (Boolean) `true` si es un propietario.
-   `m.sender.modr`: (Boolean) `true` si es un moderador.
-   `m.sender.prem`: (Boolean) `true` si es un usuario premium.
-   `m.sender.photo('image' | 'preview')`: Obtiene la URL de la foto de perfil del remitente.
-   `m.sender.desc()`: Obtiene el "info" o estado de WhatsApp del remitente.

#### `m.bot`: Información y Acciones del Bot

-   `m.bot.id`: (String) El JID del bot.
-   `m.bot.name`: (String) El nombre del bot.
-   `m.bot.admin`: (Boolean) `true` si el bot es admin del grupo (solo en grupos).
-   `m.bot.fromMe`: (Boolean) `true` si el mensaje fue enviado por el bot.
-   `m.bot.block(userJid, true|false)`: Bloquea o desbloquea a un usuario.
-   `m.bot.join(link)`: Se une a un grupo usando un enlace de invitación.

#### `m.quoted`: Información del Mensaje Citado

Este objeto solo existe si el mensaje es una respuesta a otro.
-   `m.quoted.message`: (Object) El objeto del mensaje citado.
-   `m.quoted.sender.id`: (String) El JID de quien envió el mensaje original.
-   `m.quoted.download()`: (Function) Descarga el adjunto del mensaje citado.

#### Helpers: `m.reply`, `m.react`, `m.sms`

Estas son funciones de conveniencia añadidas a `m`.
-   `m.reply(text)`: Responde al mensaje actual.
-   `m.react(emoji)`: Reacciona al mensaje. Puedes usar emojis directos ('👍') o alias ('wait', 'done', 'error').
-   `m.sms(type)`: Envía mensajes predefinidos para comprobaciones de permisos.
    -   `type` puede ser: `rowner`, `owner`, `modr`, `premium`, `group`, `private`, `admin`, `botAdmin`, `unreg`, `restrict`.

---

### 3. PROPIEDADES DEL PLUGIN (EL "ENCABEZADO")

Estas propiedades se definen en el objeto exportado y controlan cómo y cuándo se ejecuta tu plugin.

#### `command: true`
-   **Propósito:** Define que este plugin es un comando que se activa por texto.
-   **Propiedades adicionales:**
    -   `usePrefix: true | false`: Si `true`, el comando necesita un prefijo (ej: `!ping`). Si `false`, no lo necesita (ej: `ping`).
    -   `case: 'comando' | ['comando', 'alias1']`: El nombre (o nombres) que activan el comando.

#### `stubtype: true`
-   **Propósito:** Se activa por eventos de grupo (unirse, salir, cambio de nombre, etc.) en lugar de mensajes de texto.
-   **Propiedades adicionales:**
    -   `case: 'EVENTO'`: El nombre del evento de `proto.WebMessageInfo.StubType`.
-   **Contexto Adicional:** El `script` recibe un objeto adicional con `{ parameters, even }`. `parameters` contiene los JIDs de los usuarios involucrados y `even` el nombre del evento.

**Ejemplo de `case` para `stubtype`:**
- `GROUP_PARTICIPANT_ADD`: Un usuario se une o es añadido.
- `GROUP_PARTICIPANT_REMOVE`: Un usuario sale o es expulsado.
- `GROUP_PARTICIPANT_PROMOTE`: Un usuario es promovido a admin.
- `GROUP_PARTICIPANT_DEMOTE`: Un admin es degradado a miembro.
- `GROUP_CHANGE_SUBJECT`: El nombre del grupo cambia.

#### `before: true`
-   **Propósito:** Permite ejecutar código *antes* de que se procesen los comandos. Es ideal para middleware (anti-spam, baneos, logging, etc.).
-   **Propiedades adicionales:**
    -   `index: 1 | 2 | 3`: Define el orden de ejecución.
-   **Orden de Ejecución:**
    -   **`index: 1`**: Se ejecuta justo después de crear el objeto `m` básico. Aún no hay metadatos del grupo.
    -   **`index: 2`**: Se ejecuta después de cargar los metadatos del grupo (si es un grupo) y después de manejar los `stubtype`.
    -   **`index: 3`**: Se ejecuta justo antes de llamar al script del comando final. Ya tienes `m.body`, `m.command`, etc.
-   **Control de Flujo:** Puedes detener la ejecución de los siguientes plugins si en tu script de `before` haces `control.end = true;`.

#### `export`
-   **Propósito:** Permite que un plugin comparta funciones o valores con otros plugins.
-   **Cómo usarlo:**
    1.  **En el plugin que exporta:** Añade una propiedad `export` al objeto.
        ```javascript
        // /plugins/utils.js
        export default {
            export: {
                fancyLog: (text) => {
                    console.log(`✨ [LOG]: ${text} ✨`);
                }
            }
        };
        ```
    2.  **En el plugin que importa:** Usa `plugin.import('nombreDelArchivo')` dentro de tu `script`. El nombre es el del archivo sin la extensión.
        ```javascript
        // /plugins/mi-comando.js
        export default {
            command: true, usePrefix: true, case: 'log',
            script: async (m, { plugin }) => {
                const utils = plugin.import('utils'); // Importa desde utils.js
                if (utils) {
                    utils.fancyLog('Este es un mensaje desde mi-comando.js');
                    m.reply('Log enviado a la consola!');
                }
            }
        };
        ```
---

### 4. EJEMPLOS COMPLETOS

#### Comando simple: !ping
```javascript
// /plugins/ping.js
export default {
    command: true,
    usePrefix: true,
    case: 'ping',
    script: async (m) => {
        await m.react('✔️');
        await m.reply('Pong!');
    }
};
```

#### Comando con argumentos: !say
```javascript
// /plugins/say.js
export default {
    command: true,
    usePrefix: true,
    case: 'say',
    script: async (m) => {
        if (!m.text) {
            return m.reply('Por favor, escribe algo para que lo repita. Ejemplo: !say Hola mundo');
        }
        await m.reply(m.text);
    }
};
```

#### Comando de administrador: !kick
```javascript
// /plugins/kick.js
export default {
    command: true,
    usePrefix: true,
    case: 'kick',
    script: async (m, { sock }) => {
        // Validaciones
        if (!m.chat.group) return m.sms('group');
        if (!m.sender.admin) return m.sms('admin');
        if (!m.bot.admin) return m.sms('botAdmin');

        // Obtener el usuario a expulsar (de una mención o de un reply)
        const userToKick = m.message.extendedTextMessage?.contextInfo?.mentionedJid?.[0] || m.quoted?.sender.id;

        if (!userToKick) {
            return m.reply('Debes mencionar a alguien o responder a su mensaje para expulsarlo.');
        }

        try {
            await m.chat.remove(userToKick);
            await m.reply(`✅ Usuario expulsado.`);
        } catch (e) {
            await m.reply(`❌ No se pudo expulsar al usuario.`);
            console.error(e);
        }
    }
};
```

#### Evento de bienvenida (stubtype)
```javascript
// /plugins/welcome.js
export default {
    stubtype: true,
    case: 'GROUP_PARTICIPANT_ADD', // Se activa cuando alguien entra
    script: async (m, { sock, parameters }) => {
        const userJid = parameters[0]; // El JID del nuevo miembro
        const userName = (await sock['user:data'](userJid)).name || 'Usuario Nuevo';
        const groupName = m.chat.name;
        const welcomeText = `¡Bienvenid@ @${userJid.split('@')[0]} al grupo ${groupName}! 🎉`;

        // Enviar al grupo, mencionando al nuevo usuario
        await sock.sendMessage(m.chat.id, {
            text: welcomeText,
            mentions: [userJid]
        });
    }
};
```

#### Middleware de registro (before)
```javascript
// /plugins/logger-before.js
import chalk from 'chalk';

export default {
    before: true,
    index: 3, // Se ejecuta justo antes del comando
    script: async (m, { control }) => {
        if (m.isCmd) {
            console.log(
                chalk.yellow('[CMD]'),
                chalk.cyan(m.command),
                'por',
                chalk.green(m.sender.name),
                'en',
                m.chat.group ? chalk.magenta(m.chat.name) : chalk.blue('privado')
            );
        }

        // No detenemos el flujo, así que no usamos control.end
    }
};
```
