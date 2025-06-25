
# Documentación del Middleware `before`

El sistema `before` es una de las características más potentes y flexibles para el desarrollo de plugins. Actúa como un "middleware", permitiéndote ejecutar código en diferentes etapas del procesamiento de un mensaje, antes de que se ejecute el comando final. Esto es ideal para validaciones, registros (logs), sistemas de anti-spam, baneos, contadores de uso y mucho más.

##  Índice
1.  [¿Qué es un Plugin `before`?](#1-qué-es-un-plugin-before)
2.  [Estructura de un Plugin `before`](#2-estructura-de-un-plugin-before)
3.  [El Parámetro `control`: Deteniendo la Cadena de Ejecución](#3-el-parámetro-control-deteniendo-la-cadena-de-ejecución)
4.  [Los Niveles de Ejecución: `index`](#4-los-niveles-de-ejecución-index)
    -   [index: 1 (Ejecución Temprana)](#index-1-ejecución-temprana)
    -   [index: 2 (Post-Eventos y Metadatos)](#index-2-post-eventos-y-metadatos)
    -   [index: 3 (Pre-Comando)](#index-3-pre-comando)
5.  [Casos de Uso y Ejemplos Prácticos](#5-casos-de-uso-y-ejemplos-prácticos)
    -   [Ejemplo 1: Bloquear usuarios baneados (`index: 1`)](#ejemplo-1-bloquear-usuarios-baneados-index-1)
    -   [Ejemplo 2: Registrar todos los mensajes en consola (`index: 2`)](#ejemplo-2-registrar-todos-los-mensajes-en-consola-index-2)
    -   [Ejemplo 3: Sistema de Cooldown para comandos (`index: 3`)](#ejemplo-3-sistema-de-cooldown-para-comandos-index-3)
    -   [Ejemplo 4: Ignorar mensajes de grupos baneados (`index: 1`)](#ejemplo-4-ignorar-mensajes-de-grupos-baneados-index-1)

---

## 1. ¿Qué es un Plugin `before`?

Es un plugin que no se activa por un comando específico, sino que **se ejecuta para cada mensaje que el bot procesa**. Su propósito es "interceptar" el mensaje en un punto específico de su ciclo de vida para realizar una acción o validación.

Pueden existir múltiples plugins `before`, y se ejecutarán en el orden definido por su propiedad `index`.

---

## 2. Estructura de un Plugin `before`

La estructura es similar a otros plugins, pero con dos propiedades clave: `before: true` y `index`.

```javascript
// /plugins/mi-middleware.js
export default {
    // Indica que este plugin es un middleware
    before: true,

    // Define en qué etapa se ejecutará (1, 2, o 3)
    index: 1,

    // La función que se ejecutará
    script: async (m, { sock, control }) => {
        // Tu lógica de middleware aquí
        console.log(`[Middleware index 1] Mensaje recibido de ${m.sender.id}`);

        // Puedes detener la ejecución de otros plugins si es necesario
        // if (condicion) {
        //   control.end = true;
        //   return;
        // }
    }
};
```
-   **m**: El objeto de mensaje. Su contenido varía según el `index`.
-   **sock**: El objeto de conexión de Baileys.
-   **control**: Un objeto especial para manejar el flujo de ejecución.

---

## 3. El Parámetro `control`: Deteniendo la Cadena de Ejecución

El objeto `control` se pasa al `script` de un plugin `before` y tiene una propiedad fundamental:

-   **control.end** `(Boolean)`

Si dentro de tu script asignas `control.end = true`, **toda la ejecución posterior de plugins para ese mensaje se detendrá**. Esto incluye otros plugins `before` con un `index` mayor y, por supuesto, el comando final.

Esto es extremadamente útil para implementar sistemas de bloqueo. Si un usuario está baneado, un middleware de `index: 1` puede detectarlo, enviar un mensaje de "Estás baneado" y luego establecer `control.end = true` para que el bot no procese nada más de ese usuario.

---

## 4. Los Niveles de Ejecución: `index`

El `index` determina en qué punto del ciclo de vida del mensaje se ejecutará tu código.

### `index: 1` (Ejecución Temprana)

-   **Cuándo se ejecuta**: Justo después de que el objeto `m` básico es creado. Es el primer punto de control.
-   **Estado del objeto `m`**:
    -   **Disponible**: `m.message`, `m.chat.id`, `m.chat.group`, `m.sender.id`, `m.sender.name`, roles básicos del remitente.
    -   **NO disponible**: Metadatos del grupo (`m.chat.name`, `m.chat.participants`, etc.), `m.body`, `m.command`, `m.text`, `m.isCmd`.
-   **Caso de uso ideal**:
    -   Validaciones que no dependen del contenido del mensaje, como baneos de usuario o de chat.
    -   Sistemas de anti-flood que cuentan mensajes por segundo por usuario/chat.

### `index: 2` (Post-Eventos y Metadatos)

-   **Cuándo se ejecuta**: Después de que se han procesado los plugins `stubtype` (eventos de grupo) y después de que se han cargado los metadatos del grupo (si aplica).
-   **Estado del objeto `m`**:
    -   **Disponible**: Todo lo de `index: 1`, más los metadatos del grupo (`m.chat.name`, `m.chat.admins`, etc.).
    -   **NO disponible**: `m.body`, `m.command`, `m.text`, `m.isCmd`.
-   **Caso de uso ideal**:
    -   Logging general de mensajes, ya que tienes la información del grupo si es necesario.
    -   Acciones que dependen del nombre del grupo o de si el remitente es admin, pero que no necesitan analizar el texto.

### `index: 3` (Pre-Comando)

-   **Cuándo se ejecuta**: Justo antes de que el bot intente ejecutar un plugin de comando. Es el último punto de control.
-   **Estado del objeto `m`**:
    -   **Disponible**: **Todo**. El objeto `m` está completamente populado, incluyendo `m.body`, `m.command`, `m.text`, y `m.isCmd`.
-   **Caso de uso ideal**:
    -   Validaciones específicas de comandos (ej: "Este comando solo funciona en grupos").
    -   Sistemas de cooldown o economía que se aplican solo a comandos.
    -   Modificar el comportamiento de un comando (ej: redirigir un comando a otro).
    -   Registrar únicamente los comandos ejecutados.

---

## 5. Casos de Uso y Ejemplos Prácticos

### Ejemplo 1: Bloquear usuarios baneados (`index: 1`)

Este middleware revisa la base de datos en una etapa temprana y detiene todo si el usuario está baneado.

```javascript
// /plugins/ban-checker.js
import $base from '../library/fun.makeDBase.js';

export default {
    before: true,
    index: 1,
    script: async (m, { control }) => {
        const db = await $base.open('system:BUC');
        const user = db.data['@users'][m.sender.id];

        if (user && user.banned) {
            console.log(`Usuario baneado ${m.sender.id} intentó usar el bot.`);
            control.end = true; // Detiene toda ejecución posterior
            return; // No es necesario enviar mensaje, simplemente se ignora.
        }
    }
};
```

### Ejemplo 2: Registrar todos los mensajes en consola (`index: 2`)

Ideal para depuración. Usamos `index: 2` para poder mostrar el nombre del grupo.

```javascript
// /plugins/message-logger.js
import chalk from 'chalk';

export default {
    before: true,
    index: 2,
    script: async (m, { control }) => {
        const chatInfo = m.chat.group 
            ? `Grupo: ${chalk.magenta(m.chat.name)}` 
            : `Privado: ${chalk.blue(m.sender.name)}`;
            
        // En este punto, m.body no existe, así que inspeccionamos el mensaje crudo
        const body = m.message.message?.conversation || m.message.message?.extendedTextMessage?.text || '[Mensaje no textual]';

        console.log(`[MSG][${m.sender.name}] en [${chatInfo}]: ${body}`);
    }
};
```

### Ejemplo 3: Sistema de Cooldown para comandos (`index: 3`)

Este middleware evita que los usuarios usen comandos demasiado rápido. Usamos `index: 3` porque solo nos interesa actuar sobre los comandos.

```javascript
// /plugins/cooldown.js
const userCooldowns = new Map();

export default {
    before: true,
    index: 3,
    script: async (m, { control }) => {
        // Solo aplicar cooldown a comandos
        if (!m.isCmd) return;

        const cooldownTime = 5000; // 5 segundos
        const userId = m.sender.id;
        const now = Date.now();
        const lastCommandTime = userCooldowns.get(userId) || 0;

        if (now - lastCommandTime < cooldownTime) {
            const timeLeft = ((cooldownTime - (now - lastCommandTime)) / 1000).toFixed(1);
            m.reply(`⏳ Por favor, espera ${timeLeft} segundos antes de usar otro comando.`);
            control.end = true; // Detiene la ejecución del comando
            return;
        }

        // Si pasa la validación, actualizamos el tiempo del último comando
        userCooldowns.set(userId, now);
    }
};
```

### Ejemplo 4: Ignorar mensajes de grupos baneados (`index: 1`)
```javascript
// /plugins/group-ban-checker.js
import $base from '../library/fun.makeDBase.js';

export default {
    before: true,
    index: 1,
    script: async (m, { control }) => {
        if (!m.chat.group) return; // Solo aplica a grupos

        const db = await $base.open('system:BUC');
        const chat = db.data['@chats'][m.chat.id];

        if (chat && chat.banned) {
            console.log(`Mensaje ignorado del grupo baneado: ${m.chat.id}`);
            control.end = true; // Detiene todo
            return;
        }
    }
};
```
