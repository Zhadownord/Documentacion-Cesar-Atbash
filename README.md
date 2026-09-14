# Diwan al-Kindi — despliegue en Vercel con documentación protegida

Este paquete está listo para subir a [Vercel](https://vercel.com) tal cual.

## Estructura

```
.
├── index.html          → El programa (cifrado/descifrado). Público, sin contraseña.
├── documentacion.html  → Puerta de acceso: pide la contraseña, no contiene la doc.
├── api/
│   └── get-doc.js      → Función serverless: verifica la contraseña en el SERVIDOR
│                          y solo entonces devuelve el HTML completo de la doc.
├── .env.example        → Plantilla para probar localmente (no subir el .env real)
└── .gitignore
```

**Por qué es "protección real" y no solo un truco visual:** el HTML completo
de la documentación vive *dentro* de `api/get-doc.js`, como texto embebido.
Ese archivo no se sirve como página estática — solo se ejecuta en el
servidor de Vercel cuando alguien llama a `/api/get-doc`. Si la contraseña
enviada no coincide con la variable de entorno `DOC_PASSWORD`, la función
responde con un error 401 y el HTML de la documentación **nunca sale del
servidor**. No es algo que se pueda evadir abriendo las herramientas de
desarrollador, porque no hay nada que inspeccionar en el navegador hasta
que la contraseña ya fue validada del otro lado.

## 1. Configurar la contraseña en Vercel

1. Sube este proyecto a un repositorio de GitHub (o conéctalo directo con
   `vercel` CLI).
2. En [vercel.com](https://vercel.com), importa el repositorio como nuevo
   proyecto.
3. Antes de desplegar (o después, desde *Settings*), ve a
   **Project → Settings → Environment Variables** y agrega:
   - **Name:** `DOC_PASSWORD`
   - **Value:** la contraseña que quieras (ej. `criptografia2026`)
   - **Environment:** marca *Production*, *Preview* y *Development*.
4. Guarda y haz (re)deploy — las variables de entorno solo se aplican a
   despliegues nuevos, así que si la agregaste después del primer deploy,
   dispara un redeploy manual desde el dashboard.

## 2. Probarlo localmente (opcional, requiere Vercel CLI)

```bash
npm install -g vercel
cp .env.example .env      # y edita .env con tu contraseña de prueba
vercel dev
```

Esto levanta tanto el sitio estático como la función `/api/get-doc`
localmente, simulando el entorno real de Vercel.

## 3. Flujo de uso

1. Quien entra a `documentacion.html` ve solo un formulario de contraseña.
2. Al enviarla, el navegador hace `POST /api/get-doc` con la contraseña.
3. La función compara contra `DOC_PASSWORD` con una comparación en tiempo
   constante (`crypto.timingSafeEqual`) para no filtrar información por
   temporización.
4. Si es correcta, responde `200` con el HTML completo de la documentación,
   que el navegador usa para reemplazar la página (`document.write`).
5. Si es incorrecta, responde `401` y no se muestra nada del contenido.
6. Incluye un límite simple de intentos por IP (10 intentos / 10 minutos)
   como capa extra contra intentos automatizados.

## Limitaciones honestas (para tu sección de "Desarrollo" o "Conclusión")

- La contraseña viaja del navegador al servidor por HTTPS (Vercel lo hace
  automático), así que no viaja en texto plano por la red.
- El límite de intentos vive en memoria de la función; si la función
  "duerme" por inactividad (comportamiento normal de serverless) el
  contador se reinicia. Para un proyecto real se usaría una base de datos
  o un servicio como Redis/Upstash — aquí se prioriza simplicidad,
  ya que el objetivo es demostrar el concepto de verificación server-side.
- Esto protege el **contenido**, no reemplaza HTTPS ni es un sistema de
  autenticación de usuarios con cuentas — es una contraseña compartida,
  adecuada para este caso de uso académico.

## Actualizar el contenido de la documentación

El HTML que se sirve tras la contraseña correcta es el que está embebido
como texto dentro de `api/get-doc.js` (variable `DOC_HTML`). Si necesitas
editar la documentación, la forma más simple es pedir que se regenere ese
archivo a partir de una nueva versión de la página de documentación.
