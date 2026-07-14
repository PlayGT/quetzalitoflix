# Quetzalito Proxy v2 (seguro)

## Qué cambia respecto a tu versión anterior

| Antes (`proxy.js`) | Ahora |
|---|---|
| Cliente pasa `?url=http://tv.m3uts.xyz/...` en texto plano (visible con F12 / logcat) | Cliente solo pide `/api/live/905` |
| Headers reales (`X-Hash`, `X-Did`...) fijos en un archivo que también se sube al repo del front | Headers viven solo en `lib/channels.js`, en el backend |
| Segmentos reescritos como `/api/proxy?url=<url-en-claro>` | Segmentos reescritos como `/api/seg?t=<token cifrado con expiración>` |
| Cualquiera con el link del proxy puede reproducirlo desde donde sea | Puedes exigir un header secreto (`x-app-key`) para que solo tu app pueda usarlo |

## Estructura

```
api/
  live/[id].js   <- único endpoint que tu app llama: GET /api/live/905
  seg.js         <- endpoint interno para segmentos/keys (nunca lo llamas directo)
lib/
  channels.js    <- mapping id -> {url real, headers reales}. AQUÍ agregas canales.
  security.js    <- cifrado AES-256-GCM de los tokens + validación de x-app-key
  stream.js      <- lógica compartida de fetch + reescritura de playlists
```

## 1. Agregar canales

Edita `lib/channels.js` y agrega entradas al objeto `CHANNELS`:

```js
1234: {
  url: 'http://tv.m3uts.xyz/stream/secure/l_BLgRf5/1234.m3u8',
  headers: SHARED_HEADERS_TVM3UTS, // o un objeto de headers distinto si el canal es de otro origen
},
```

Este archivo **nunca** se expone al cliente porque solo se importa dentro de las funciones serverless.

## 2. Secretos (ya están en el código, no necesitas tocar Vercel)

Abre `lib/security.js` y cambia:

- `PROXY_SECRET`: cualquier string largo y aleatorio. Cifra los tokens de segmento.
- `APP_SECRET`: déjalo vacío (`''`) o pon un valor si quieres exigir que tu app mande el header `X-App-Key` para poder usar `/api/live/:id`.

Como el repo es privado, es aceptable tenerlos hardcodeados aquí en vez de usar variables de entorno de Vercel.

## 3. Deploy

Solo sube el código a tu repo privado en GitHub y conéctalo/déjalo conectado en Vercel — se despliega solo, sin pasos extra en el dashboard.

## 4. Cómo se ve desde el cliente

La app (o el navegador) solo ve:

```
GET https://tu-proyecto.vercel.app/api/live/905
```

y dentro del `.m3u8` que devuelve, todas las líneas de segmentos/keys son:

```
https://tu-proyecto.vercel.app/api/seg?t=eyJhbGc...(token opaco, cambia cada 30 min)
```

Nunca aparece `tv.m3uts.xyz` ni `X-Hash`/`X-Did`/`X-Version` en ningún punto que el cliente pueda inspeccionar.

## Notas

- Los tokens de segmento expiran a los 30 minutos (`TOKEN_TTL_SECONDS` en `lib/stream.js`). Ajusta si tus streams tardan más en cargar.
- Si en el futuro necesitas headers *distintos por canal* (no solo por origen), ya está soportado: solo pon un objeto `headers` distinto en cada entrada de `lib/channels.js`.
- Si Vercel te pide runtime de Node explícito, agrega `"runtime": "nodejs20.x"` dentro de cada función en `vercel.json` (opcional, Vercel detecta uno reciente por defecto).
