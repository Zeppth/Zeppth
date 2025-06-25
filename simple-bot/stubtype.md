
# Eventos de Grupo con `stubtype`

El sistema de `stubtype` permite a tus plugins reaccionar a eventos que ocurren dentro de un grupo de WhatsApp, en lugar de a mensajes de texto enviados por usuarios. Estos eventos incluyen cuando alguien se une o es expulsado, cuando se cambia el nombre del grupo, cuando se promueve a un administrador, y más.

## Índice
1.  [¿Qué es un Plugin `stubtype`?](#1-qué-es-un-plugin-stubtype)
2.  [Estructura de un Plugin `stubtype`](#2-estructura-de-un-plugin-stubtype)
3.  [El Contexto Adicional: `parameters` y `even`](#3-el-contexto-adicional-parameters-y-even)
4.  [Lista de Eventos (`case`) más Comunes](#4-lista-de-eventos-case-más-comunes)
5.  [Ejemplos Prácticos](#5-ejemplos-prácticos)
    -   [Ejemplo 1: Mensaje de Bienvenida (`GROUP_PARTICIPANT_ADD`)](#ejemplo-1-mensaje-de-bienvenida-group_participant_add)
    -   [Ejemplo 2: Mensaje de Despedida (`GROUP_PARTICIPANT_REMOVE`)](#ejemplo-2-mensaje-de-despedida-group_participant_remove)
    -   [Ejemplo 3: Notificar cambios de nombre de grupo (`GROUP_CHANGE_SUBJECT`)](#ejemplo-3-notificar-cambios-de-nombre-de-grupo-group_change_subject)
    -   [Ejemplo 4: Detectar cuando el bot es añadido a un grupo (`GROUP_CREATE`)](#ejemplo-4-detectar-cuando-el-bot-es-añadido-a-un-grupo-group_create)

---

## 1. ¿Qué es un Plugin `stubtype`?

Es un tipo de plugin que se ejecuta cuando WhatsApp informa sobre una acción o cambio de estado en un grupo. Estos mensajes no son textos convencionales, sino notificaciones automáticas que aparecen en el chat, como "Juan se unió al grupo" o "El administrador cambió el ícono de este grupo".

Tu bot puede "escuchar" estos eventos y ejecutar una acción personalizada, como dar la bienvenida, despedir a un usuario, registrar cambios, etc.

---

## 2. Estructura de un Plugin `stubtype`

La estructura es muy específica y se diferencia de los plugins de comando y de los middleware `before`.

```javascript
// /plugins/evento-bienvenida.js
export default {
    // Indica que este plugin reacciona a eventos de grupo
    stubtype: true,

    // Define el evento específico que activará este plugin
    case: 'GROUP_PARTICIPANT_ADD',

    // La función que se ejecutará cuando ocurra el evento
    script: async (m, { sock, parameters, even }) => {
        // Tu lógica de evento aquí
        const newUserJid = parameters[0];
        console.log(`El usuario ${newUserJid} se unió al grupo ${m.chat.id}. Evento: ${even}`);
        
        await sock.sendMessage(m.chat.id, { 
            text: `¡Bienvenido/a @${newUserJid.split('@')[0]}!`,
            mentions: [newUserJid]
        });
    }
};
```
-   **`stubtype: true`**: Es la propiedad que le dice al manejador de plugins que este archivo es para eventos.
-   **`case: 'NOMBRE_DEL_EVENTO'`**: Especifica a qué evento reaccionar. El nombre debe coincidir con los definidos en `proto.WebMessageInfo.StubType` de Baileys.
-   **`script(m, { ... })`**: La función que se ejecuta. Recibe el objeto `m` y un segundo objeto con contexto adicional.

---

## 3. El Contexto Adicional: `parameters` y `even`

Cuando un plugin `stubtype` se ejecuta, su función `script` recibe un segundo argumento con propiedades específicas para el evento:

-   **`parameters`** `(Array<String>)`
    Un array de strings que contiene los JIDs de los usuarios involucrados en el evento. El contenido de este array depende del tipo de evento.
    -   Para `GROUP_PARTICIPANT_ADD`, `GROUP_PARTICIPANT_REMOVE`, `GROUP_PARTICIPANT_PROMOTE`, etc., `parameters[0]` suele ser el JID del usuario sobre el cual se realizó la acción.

-   **`even`** `(String)`
    El nombre del evento que activó el plugin (el mismo que se define en `case`). Es útil si un mismo plugin maneja varios casos.

-   **`m`**: El objeto `m` en un plugin `stubtype` contiene información del chat (`m.chat`) y metadatos del grupo, pero no contendrá propiedades relacionadas con texto como `m.body` o `m.command`, ya que no es un mensaje de texto.

---

## 4. Lista de Eventos (`case`) más Comunes

Aquí tienes una lista de los valores más útiles que puedes usar en la propiedad `case`:

-   **`GROUP_PARTICIPANT_ADD`**: Un nuevo miembro se une al grupo o es añadido por un admin.
    -   `parameters[0]`: JID del nuevo miembro.
    -   `m.sender.id`: JID de quien lo añadió (si fue añadido). Si se unió por enlace, es el JID del nuevo miembro.

-   **`GROUP_PARTICIPANT_REMOVE`**: Un miembro es expulsado.
    -   `parameters[0]`: JID del miembro expulsado.
    -   `m.sender.id`: JID del admin que lo expulsó.

-   **`GROUP_PARTICIPANT_LEAVE`**: Un miembro abandona el grupo por su cuenta.
    -   `parameters[0]`: JID del miembro que se fue.

-   **`GROUP_PARTICIPANT_PROMOTE`**: Un miembro es promovido a administrador.
    -   `parameters[0]`: JID del miembro promovido.
    -   `m.sender.id`: JID del admin que lo promovió.

-   **`GROUP_PARTICIPANT_DEMOTE`**: Un administrador es degradado a miembro común.
    -   `parameters[0]`: JID del admin degradado.
    -   `m.sender.id`: JID del admin que lo degradó.

-   **`GROUP_CHANGE_SUBJECT`**: El nombre (asunto) del grupo es modificado.
    -   `m.sender.id`: JID del admin que cambió el nombre.

-   **`GROUP_CHANGE_ICON`**: El ícono (foto) del grupo es modificado.
    -   `m.sender.id`: JID del admin que cambió la foto.

-   **`GROUP_CREATE`**: El bot es añadido a un nuevo grupo.
    -   `m.sender.id`: JID de la persona que creó el grupo y añadió al bot.
    -   `m.chat.id`: El JID del nuevo grupo.

-   **`E2E_ENCRYPTION`**: La encriptación se ha restablecido. Suele ocurrir al reinstalar WhatsApp.

---

## 5. Ejemplos Prácticos

### Ejemplo 1: Mensaje de Bienvenida (`GROUP_PARTICIPANT_ADD`)
Este es el uso más común. Da la bienvenida a los nuevos miembros y los menciona.

```javascript
// /plugins/welcome.js
export default {
    stubtype: true,
    case: 'GROUP_PARTICIPANT_ADD',
    script: async (m, { sock, parameters }) => {
        const newUserJid = parameters[0];
        
        // Evitar que el bot se dé la bienvenida a sí mismo
        if (newUserJid === m.bot.id) return;

        const groupName = m.chat.name;
        const welcomeMessage = `👋 ¡Hola @${newUserJid.split('@')[0]}! Te damos la bienvenida a ${groupName}. ¡Esperamos que disfrutes tu estancia!`;

        try {
            const profilePicUrl = await sock['user:data'](newUserJid).photo('image');
            await sock.sendMessage(m.chat.id, {
                image: { url: profilePicUrl },
                caption: welcomeMessage,
                mentions: [newUserJid]
            });
        } catch (e) {
            // Si falla la obtención de la foto, enviar solo texto
            await sock.sendMessage(m.chat.id, {
                text: welcomeMessage,
                mentions: [newUserJid]
            });
        }
    }
};
```

### Ejemplo 2: Mensaje de Despedida (`GROUP_PARTICIPANT_REMOVE`)
Notifica cuando un usuario es expulsado del grupo.

```javascript
// /plugins/goodbye.js
export default {
    stubtype: true,
    case: 'GROUP_PARTICIPANT_REMOVE',
    script: async (m, { sock, parameters }) => {
        const removedUserJid = parameters[0];
        const adminJid = m.sender.id;

        const removedUserName = removedUserJid.split('@')[0];
        const adminName = (await sock['user:data'](adminJid)).name;

        const goodbyeMessage = `Adiós @${removedUserName}. Fue expulsado por ${adminName}.`;
        
        await sock.sendMessage(m.chat.id, {
            text: goodbyeMessage,
            mentions: [removedUserJid]
        });
    }
};
```

### Ejemplo 3: Notificar cambios de nombre de grupo (`GROUP_CHANGE_SUBJECT`)
Envía un mensaje personalizado cuando el nombre del grupo cambia.

```javascript
// /plugins/notify-subject-change.js
export default {
    stubtype: true,
    case: 'GROUP_CHANGE_SUBJECT',
    script: async (m, { sock }) => {
        const adminName = m.sender.name;
        const newGroupName = m.chat.name; // m.chat ya tiene el nuevo nombre

        const message = `ⓘ El administrador ${adminName} ha cambiado el nombre del grupo a: *${newGroupName}*`;
        await m.reply(message);
    }
};
```

### Ejemplo 4: Detectar cuando el bot es añadido a un grupo (`GROUP_CREATE`)
Ideal para presentarse y, opcionalmente, salir si el grupo no es autorizado.

```javascript
// /plugins/bot-on-join.js
export default {
    stubtype: true,
    case: 'GROUP_CREATE',
    script: async (m, { sock }) => {
        const creatorJid = m.sender.id;
        const groupName = m.chat.name;

        // Lista de usuarios autorizados para añadir el bot
        const authorizedUsers = ['1234567890@s.whatsapp.net', '0987654321@s.whatsapp.net'];

        if (!authorizedUsers.includes(creatorJid)) {
            await m.reply(`Hola, ${groupName}. No tengo permiso para estar aquí. Me iré ahora. Contacta a mi dueño para obtener permiso.`);
            await sock.groupLeave(m.chat.id);
            return;
        }

        await m.reply(`¡Gracias por añadirme a ${groupName}! Usa ` + ``/!menu`` + ` para ver lo que puedo hacer.`);
    }
};
```
