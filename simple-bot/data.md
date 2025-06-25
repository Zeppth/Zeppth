
# Objeto `m` (o `data`)

El objeto `m` es el parámetro más crucial para cualquier plugin. Contiene todo el contexto de un mensaje entrante, procesado y enriquecido con funciones y datos útiles para facilitar el desarrollo. A continuación se detalla cada una de sus partes.

##  Índice
1.  [Propiedades de Alto Nivel](#1-propiedades-de-alto-nivel)
2.  [Métodos de Ayuda (Helpers)](#2-métodos-de-ayuda-helpers)
3.  [El Objeto `m.sender`](#3-el-objeto-msender)
4.  [El Objeto `m.chat`](#4-el-objeto-mchat)
5.  [El Objeto `m.bot`](#5-el-objeto-mbot)
6.  [El Objeto `m.quoted`](#6-el-objeto-mquoted)

---

## 1. Propiedades de Alto Nivel

Estas son las propiedades más directas y comunes que usarás en tus plugins de comando.

-   **m.message** `(Object)`
    El objeto de mensaje crudo y completo recibido de Baileys. Contiene toda la información sin procesar.

-   **m.body** `(String)`
    El contenido de texto principal del mensaje, ya extraído. Funciona para textos, descripciones de imágenes/videos y respuestas a botones.
    *Ejemplo: Para un mensaje "¡hola mundo!", `m.body` sería "¡hola mundo!".*

-   **m.command** `(String)`
    Si el mensaje es un comando, esta propiedad contiene el nombre del comando en minúsculas.
    *Ejemplo: Para "!Ping", `m.command` sería "ping".*

-   **m.text** `(String | null)`
    El resto del texto del mensaje después del comando. Si no hay texto adicional, es `null`.
    *Ejemplo: Para "!say Hola a todos", `m.text` sería "Hola a todos".*

-   **m.args** `(Array<String>)`
    Un array con cada "palabra" del `m.text`, separadas por espacios.
    *Ejemplo: Para "!add 5 10", `m.args` sería `['5', '10']`.*

-   **m.isCmd** `(Boolean)`
    Es `true` si el sistema ha identificado el mensaje como un comando válido (según los plugins existentes).

-   **m.plugin** `(Object | null)`
    Si `m.isCmd` es `true`, esta propiedad contiene el objeto completo del plugin que se va a ejecutar.

-   **m.tag** `(Array<String>)`
    Un array con los valores de las "etiquetas" extraídas del cuerpo del mensaje. Las etiquetas tienen el formato `tag=valor`.
    *Ejemplo: Para "!buscar tag=imagen tag=hd gatitos", `m.tag` sería `['imagen', 'hd']` y `m.body` se limpiaría a "!buscar gatitos".*

---

## 2. Métodos de Ayuda (Helpers)

Funciones de conveniencia añadidas directamente al objeto `m`.

-   **m.reply(text, [footer])**
    Responde al mensaje actual de forma estilizada usando un `interactiveMessage`.
    -   **text** `(String)`: El texto que se enviará.
    -   **footer** `(String, opcional)`: Un texto de pie de página. Por defecto es el nombre del bot.

-   **m.react(emoji)**
    Reacciona al mensaje actual.
    -   **emoji** `(String)`: Puede ser un emoji directo ('✅') o un alias predefinido:
        -   `'wait'` ➜ '⌛'
        -   `'done'` ➜ '✔️'
        -   `'error'` ➜ '✖️'

-   **m.sms(type)**
    Envía un mensaje de sistema predefinido, útil para validaciones de permisos.
    -   **type** `(String)`: El tipo de mensaje. Valores posibles:
        -   `'rowner'`: "Este comando solo puede ser utilizado por el *dueño*".
        -   `'owner'`: "Este comando solo puede ser utilizado por un *propietario*".
        -   `'modr'`: "Este comando solo puede ser utilizado por un *moderador*".
        -   `'premium'`: "Esta solicitud es solo para usuarios *premium*".
        -   `'group'`: "Este comando solo se puede usar en *grupos*".
        -   `'private'`: "Este comando solo se puede usar por *chat privado*".
        -   `'admin'`: "Este comando solo puede ser usado por los *administradores del grupo*".
        -   `'botAdmin'`: "El bot necesita *ser administrador* para usar este comando".
        -   `'unreg'`: Mensaje para registrarse.
        -   `'restrict'`: "Esta función está desactivada".

-   **m.download([message], [type])**
    Descarga el contenido multimedia del mensaje.
    -   **message** `(Object, opcional)`: El objeto del mensaje del que se descargará. Por defecto, `m.message`.
    -   **type** `(String, opcional)`: El formato de salida. Por defecto, `'buffer'`.
    *Ejemplo: `const imageBuffer = await m.download();`*

---

## 3. El Objeto `m.sender`

Información sobre la persona que envió el mensaje.

-   **m.sender.id** `(String)`
    El JID (Jabber ID) del remitente (ej: `'1234567890@s.whatsapp.net'`).

-   **m.sender.name** `(String)`
    El nombre de perfil (pushName) del remitente.

-   **m.sender.number** `(String)`
    El número de teléfono del remitente, sin el sufijo `@s.whatsapp.net`.

-   **m.sender.bot** `(Boolean)`
    Es `true` si el remitente es el propio bot.

-   **m.sender.admin** `(Boolean)`
    (Solo en grupos) Es `true` si el remitente es administrador del grupo.

-   **m.sender.rowner** `(Boolean)`
-   **m.sender.owner** `(Boolean)`
-   **m.sender.modr** `(Boolean)`
-   **m.sender.prem** `(Boolean)`
    Roles asignados al usuario desde la base de datos (`system:BUC`) y el archivo `settings.json`.

-   **async m.sender.photo([type])**
    Obtiene la URL de la foto de perfil del remitente.
    -   **type** `(String, opcional)`: Puede ser `'image'` (alta resolución) o `'preview'` (baja resolución).

-   **async m.sender.desc()**
    Obtiene el estado o "info" de WhatsApp del remitente.

---

## 4. El Objeto `m.chat`

Información sobre el chat donde se recibió el mensaje.

-   **m.chat.id** `(String)`
    El JID del chat. Puede ser un JID de usuario (`@s.whatsapp.net`) o de grupo (`@g.us`).

-   **m.chat.group** `(Boolean)`
    Un getter que devuelve `true` si el chat es un grupo.

### Propiedades y Métodos Solo para Grupos
Estas propiedades y métodos solo están disponibles si `m.chat.group` es `true`.

#### Propiedades del Grupo:
-   **m.chat.name** `(String)`: Nombre del grupo.
-   **m.chat.desc** `(String)`: Descripción del grupo.
-   **m.chat.participants** `(Array)`: Array de objetos de los participantes.
-   **m.chat.admins** `(Array<String>)`: Array con los JIDs de los administradores.
-   **m.chat.owner** `(String)`: JID del creador del grupo.
-   **m.chat.size** `(Number)`: Cantidad de participantes.
-   **m.chat.created** `(Number)`: Timestamp de la creación del grupo.

#### Métodos de Moderación:
-   **async m.chat.promote(userJid)**: Promueve a un miembro a administrador.
-   **async m.chat.demote(userJid)**: Degrada a un administrador a miembro.
-   **async m.chat.remove(userJid)**: Expulsa a un miembro del grupo.
-   **async m.chat.add(userJid)**: Añade a un miembro al grupo.

#### Métodos de Configuración:
-   **async m.chat.settings.lock(boolean)**: Si es `true`, solo los admins pueden enviar mensajes.
-   **async m.chat.settings.announce(boolean)**: Si es `true`, el grupo se convierte en un canal de anuncios (es un alias de `lock`).
-   **async m.chat.settings.member_add(boolean)**: Si es `true`, todos los miembros pueden añadir a otros.
-   **async m.chat.settings.join_approval(boolean)**: Si es `true`, se activa la aprobación de nuevos miembros.

#### Métodos de Actualización:
-   **async m.chat.update.name(new_name)**: Cambia el nombre del grupo.
-   **async m.chat.update.desc(new_desc)**: Cambia la descripción del grupo.
-   **async m.chat.update.photo(image, [type])**: Cambia la foto del grupo. `image` puede ser un Buffer o URL.

#### Métodos de Invitación:
-   **async m.chat.invite.code()**: Obtiene el código de invitación.
-   **async m.chat.invite.link()**: Obtiene el enlace de invitación completo.
-   **async m.chat.invite.revoke()**: Revoca y genera un nuevo enlace de invitación.

---

## 5. El Objeto `m.bot`

Información y acciones relacionadas con la identidad del bot.

-   **m.bot.id** `(String)`: El JID del bot.
-   **m.bot.name** `(String)`: El nombre de perfil del bot.
-   **m.bot.number** `(String)`: El número de teléfono del bot.
-   **m.bot.fromMe** `(Boolean)`: Es `true` si el mensaje fue enviado desde la cuenta del bot.
-   **m.bot.admin** `(Boolean)`: (Solo en grupos) Es `true` si el bot es administrador.

#### Métodos de Acción del Bot:
-   **async m.bot.join(link)**: Se une a un grupo a través de un enlace de invitación.
-   **async m.bot.block(userJid, boolean)**: Bloquea (`true`) o desbloquea (`false`) a un usuario.
-   **async m.bot.mute(chatId, boolean, [time])**: Silencia (`true`) o des-silencia (`false`) un chat.
-   **async m.bot.update.name(new_name)**: Cambia el nombre de perfil del bot.
-   **async m.bot.update.desc(new_desc)**: Cambia el estado/info del bot.
-   **async m.bot.update.photo(image)**: Cambia la foto de perfil del bot.

---

## 6. El Objeto `m.quoted`

Este objeto **solo existe si el mensaje actual es una respuesta (reply) a otro mensaje**. Siempre verifica su existencia con `if (m.quoted)`.

-   **m.quoted.message** `(Object)`: El objeto del mensaje citado, con su propia estructura de Baileys. Para obtener el texto de un mensaje citado simple, usarías `m.quoted.message.conversation`.

-   **m.quoted.sender** `(Object)`: Un objeto con información del remitente del mensaje citado.
    -   **m.quoted.sender.id** `(String)`: JID del remitente original.
    -   **m.quoted.sender.number** `(String)`: Número del remitente original.
    -   **async m.quoted.sender.photo([type])**
    -   **async m.quoted.sender.desc()**

-   **async m.quoted.download([type])**
    Descarga el contenido multimedia (si lo hay) del mensaje citado.
