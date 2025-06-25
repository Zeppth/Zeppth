
# Sistema de `export` e `import`

El sistema de `export` es una característica fundamental para crear un código limpio, modular y reutilizable. Te permite definir funciones, objetos o valores en un plugin y utilizarlos en cualquier otro, evitando la duplicación de código y centralizando la lógica común.

## Índice
1.  [¿Para qué sirve el sistema de `export`?](#1-para-qué-sirve-el-sistema-de-export)
2.  [Cómo Exportar: La Propiedad `export`](#2-cómo-exportar-la-propiedad-export)
3.  [Cómo Importar: El Método `plugin.import()`](#3-cómo-importar-el-método-pluginimport)
4.  [Punto Clave: El Nombre del Export vs. El Nombre del Archivo](#4-punto-clave-el-nombre-del-export-vs-el-nombre-del-archivo)
5.  [Ejemplos Prácticos](#5-ejemplos-prácticos)
    -   [Ejemplo 1: Creando una Librería de Utilidades](#ejemplo-1-creando-una-librería-de-utilidades)
    -   [Ejemplo 2: Exportando una única función](#ejemplo-2-exportando-una-única-función)
    -   [Ejemplo 3: Compartiendo una variable o configuración](#ejemplo-3-compartiendo-una-variable-o-configuración)

---

## 1. ¿Para qué sirve el sistema de `export`?

Imagina que tienes una función que formatea texto de una manera especial, o una que se conecta a una base de datos externa. En lugar de copiar y pegar esa función en cada plugin que la necesite, puedes:
1.  Definirla una sola vez en un plugin (ej: `utils.js`).
2.  **Exportarla** con un nombre único.
3.  **Importarla** en cualquier otro plugin que la requiera.

**Ventajas:**
-   **Reutilización de código:** Escribe una vez, úsalo en todas partes.
-   **Mantenimiento sencillo:** Si necesitas cambiar la función, solo lo haces en un archivo.
-   **Código más limpio:** Tus plugins de comandos se centran en su lógica principal, no en funciones auxiliares.

---

## 2. Cómo Exportar: La Propiedad `export`

Para compartir código desde un plugin, debes añadir una propiedad llamada `export` al objeto que exportas por defecto. El valor de esta propiedad **debe ser un objeto**.

Las **claves** de este objeto serán los **nombres que usarás para importar**, y los **valores** serán lo que quieres compartir (una función, un objeto, una variable, etc.).

```javascript
// /plugins/mis-utilidades.js

// Una función que queremos compartir
const calcularImpuesto = (precio) => {
    return precio * 1.21;
};

// Otra función
const saludar = (nombre) => {
    return `Hola, ${nombre}, ¡bienvenido!`;
}

export default {
    // Esta propiedad es la que importa
    export: {
        // 'calculadora' es el nombre con el que importaremos el objeto
        calculadora: {
            impuesto: calcularImpuesto
        },
        // 'saludos' es el nombre con el que importaremos la función
        saludos: saludar
    }
    // Este plugin no necesita un 'script' si solo es para exportar
};
```

---

## 3. Cómo Importar: El Método `plugin.import()`

Dentro del `script` de cualquier otro plugin, tienes acceso al manejador de plugins a través del segundo argumento (`{ sock, plugin }`). Este manejador tiene el método `import()`.

-   **`plugin.import('nombreDelExport')`**: Este método busca en todos los exports cargados una clave que coincida con `'nombreDelExport'` y devuelve su valor.

```javascript
// /plugins/tienda.js

export default {
    command: true,
    usePrefix: true,
    case: 'precio',

    script: async (m, { plugin }) => {
        // Importamos los objetos/funciones que necesitamos
        const calc = plugin.import('calculadora');
        const saludarFn = plugin.import('saludos');

        if (!m.text || isNaN(m.text)) {
            return m.reply('Por favor, ingresa un precio numérico.');
        }

        const precioBase = parseFloat(m.text);
        const precioFinal = calc.impuesto(precioBase);
        
        const saludo = saludarFn(m.sender.name);
        
        const respuesta = `${saludo}\nEl precio final con impuestos es: ${precioFinal.toFixed(2)}`;
        await m.reply(respuesta);
    }
};
```

---

## 4. Punto Clave: El Nombre del Export vs. El Nombre del Archivo

Es **muy importante** entender que cuando importas, **no usas el nombre del archivo**, sino la **clave que definiste dentro del objeto `export`** del otro plugin.

-   **Incorrecto**: `plugin.import('mis-utilidades')` ⬅️ ¡ESTO NO FUNCIONARÁ!
-   **Correcto**: `plugin.import('calculadora')` y `plugin.import('saludos')` ✔️

Puedes tener múltiples claves de exportación en un solo archivo.

---

## 5. Ejemplos Prácticos

### Ejemplo 1: Creando una Librería de Utilidades

Este es el caso más común: agrupar varias funciones útiles en un solo objeto.

**Archivo que exporta: `/plugins/utils.js`**
```javascript
const formatNumber = (num) => {
    return num.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".");
}

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

export default {
    export: {
        // Exportamos un objeto llamado 'helpers' que contiene nuestras funciones
        helpers: {
            format: formatNumber,
            wait: sleep
        }
    }
};
```

**Archivo que importa: `/plugins/donar.js`**
```javascript
export default {
    command: true,
    case: 'donar',
    script: async (m, { plugin }) => {
        const utils = plugin.import('helpers'); // Importamos el objeto 'helpers'

        await m.react('⌛');
        await utils.wait(2000); // Usamos la función importada

        const donacion = 50000;
        const texto = `Gracias por tu donación de ${utils.format(donacion)}!`;
        await m.reply(texto);
    }
};
```

### Ejemplo 2: Exportando una única función

Si solo quieres compartir una función.

**Archivo que exporta: `/plugins/logger.js`**
```javascript
import chalk from 'chalk';

const logAction = (user, action) => {
    console.log(chalk.bgBlue('[LOG]'), `El usuario ${user} realizó la acción: ${action}`);
};

export default {
    export: {
        // El nombre de exportación es 'logAction'
        logAction: logAction
    }
};
```

**Archivo que importa: `/plugins/kick.js`**
```javascript
export default {
    command: true, case: 'kick',
    script: async (m, { plugin }) => {
        const log = plugin.import('logAction'); // Importamos la función directamente
        // ...lógica para expulsar...
        if (log) {
            log(m.sender.name, 'expulsó a un usuario');
        }
        await m.reply('Usuario expulsado.');
    }
};
```

### Ejemplo 3: Compartiendo una variable o configuración

También puedes exportar valores estáticos como arrays o configuraciones.

**Archivo que exporta: `/plugins/config-commands.js`**
```javascript
const premiumCommands = ['play-hd', 'sticker-pro', 'backup'];

export default {
    export: {
        premiumCmds: premiumCommands
    }
};
```

**Archivo que importa (un middleware `before`): `/plugins/premium-check.js`**
```javascript
export default {
    before: true,
    index: 3,
    script: async (m, { plugin, control }) => {
        if (!m.isCmd) return;

        const listaPremium = plugin.import('premiumCmds');

        if (listaPremium && listaPremium.includes(m.command) && !m.sender.prem) {
            m.sms('premium');
            control.end = true; // Detiene la ejecución si no es premium
        }
    }
};
```
