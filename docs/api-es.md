# Guía de desarrollo de Senzhi AI · API, Feishu Base y ComfyUI

Automatiza una tarea de catálogo a partir de una fila con un SKU y una foto. Esta guía muestra cómo elegir un modelo, encargar una imagen con fondo blanco y guardar el original para devolver el resultado a la ficha de ese mismo artículo.

[Gestionar claves de API](https://www.senzhiai.com/workspace.php?tool=open_api) · [Guía del producto y de la generación por lotes](es.md) · [English](api-en.md)

## 1. Consulta el catálogo de tu cuenta

Entra en el espacio de trabajo y crea una clave en el apartado de acceso a la API. Los ejemplos de Bash y curl toman la clave de la variable de entorno `SENZHI_API_KEY` de tu servidor. Envía la clave en la cabecera `Authorization: Bearer <key>` o `X-API-Key`.

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  'https://www.senzhiai.com/api/open_models.php?type=image'
```

El filtro `type` acepta `image`, `text` o `video`. Omítelo para consultar todos los tipos. La consulta es gratuita. Dentro de `models`, elige un modelo y revisa sus campos `pricing` y `request_example`. El identificador de Senzhi se indica en `model_id`, por ejemplo `db:123`. Para buscar por nombre, usa `model_name` en lugar del identificador y copia el nombre tal como aparece en el catálogo, respetando mayúsculas y minúsculas. Si dos entradas comparten nombre, recibirás `409 AMBIGUOUS_MODEL`. En ese caso, indica el ID.

## 2. Solicita una foto para el catálogo

Guarda el siguiente JSON como `request.json`. Sustituye la URL de ejemplo por un enlace HTTPS directo a tu propia imagen que Senzhi pueda leer. El orden de `images` define qué foto es la imagen 1 o la imagen 2. Si cambias de modelo, incorpora los campos obligatorios de su `request_example`.

```json
{
  "model_name": "nano-banana-pro",
  "prompt": "Crea una foto frontal sobre fondo blanco de la chaqueta de punto color crema de la imagen 1. Conserva el cuello, los botones y el canalé de los puños.",
  "images": [
    "https://example.com/products/knit-01.jpg"
  ],
  "idempotency_key": "catalog-knit-01-white-v1"
}
```

```bash
curl --silent --show-error \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @request.json \
  'https://www.senzhiai.com/api/open_generate.php'
```

Cada petición crea una tarea y `n` vale 1. Guarda el `request_id` de la respuesta, por ejemplo `{"success":true,"request_id":"req_example","status":"queued"}`. Usa un nuevo `idempotency_key` para cada trabajo nuevo. Si pierdes la conexión antes de recibir el ID, vuelve a enviar **el mismo cuerpo, con la misma clave de API y la misma clave de idempotencia** para recuperar el trabajo que pudo haberse aceptado.

## 3. Consulta la tarea y guarda el original

Sustituye `req_example` por el ID recibido. Realiza la consulta con la misma clave de API con la que creaste la tarea.

```bash
curl --silent --show-error --get \
  -H "Authorization: Bearer $SENZHI_API_KEY" \
  --data-urlencode 'request_id=req_example' \
  'https://www.senzhiai.com/api/open_result.php'
```

| Valor de `status` en el JSON | Qué hacer |
| --- | --- |
| `pending` / `queued` / `processing` | Espera de 5 a 10 segundos y consulta de nuevo. Si recibes `Retry-After` o `retry_after`, respeta el plazo indicado. |
| `success` | Lee `result.url` para imágenes y vídeos, o `result.content` para texto. Descarga enseguida el archivo original conservando todos los parámetros de la URL. |
| `failed` | Registra `error.code` y `error.message` y revisa el motivo. |

Si una consulta supera el tiempo de espera o devuelve `503 MEDIA_UNAVAILABLE`, continúa consultando el ID original. `410 MEDIA_EXPIRED` indica que ha vencido el plazo de conservación. Para descargar, envía únicamente la URL de `result.url` al servidor del archivo. Utiliza la clave de Senzhi exclusivamente en las peticiones al dominio de su API.

## Cómo elegir los parámetros de cada modelo

- **Imagen.** Consulta `parameter_combinations` y `parameters` para elegir una combinación de resolución, proporción, calidad y formato. Para generar en 4K, confirma que la combinación elegida ofrece esa resolución.
- **Texto.** Incluye en `prompt` los datos verificados del producto, el idioma y la extensión. Puedes usar `system_prompt` para especificar el tono de la marca. Antes de adjuntar fotos, comprueba que el modelo admite imágenes y cuántas permite.
- **Vídeo.** Podrás solicitarlo cuando el catálogo `type=video` de tu cuenta incluya un modelo. El 6 de noviembre de 2026, los catálogos consultados tenían la lista de vídeo vacía. MiniMax H3 y Agnes Video 2.5 estaban disponibles en el espacio de trabajo web. Para un modelo de vídeo de la API, consulta `video_capabilities`, conserva `options.pricing_version` del ejemplo y calcula el coste multiplicando la tarifa de la resolución elegida por la duración en segundos.

### Redacta un punto de venta para una ficha en inglés

Guarda este cuerpo como `request.json` y reutiliza las llamadas de envío y consulta del ejemplo de imagen. Sustituye la clave de idempotencia por una propia de esta nueva tarea y consérvala si necesitas reintentar el mismo envío. Cuando el estado sea `success`, lee el texto en `result.content`.

```json
{
  "model_name": "gpt-6.1-sol",
  "prompt": "Escribe en inglés una frase de hasta 20 palabras para una ficha de producto. Usa solo estos hechos verificados: es una chaqueta de punto color crema y tiene cuatro botones delanteros.",
  "idempotency_key": "catalog-knit-01-copy-en-v1"
}
```

### Nombres de modelos verificados el 6 de noviembre de 2026

Para usar `model_name`, copia el nombre con las mayúsculas, espacios y paréntesis del catálogo. Antes de enviar una tarea, consulta el catálogo actualizado de tu cuenta.

| Tipo | Nombre en el catálogo |
| --- | --- |
| Imagen | `nano-banana-pro` · `nano-banana-pro-vip（谷歌官方）` · `Nano Banana Pro【VIP】` |
| Imagen | `nano-banana-2` · `nano-banana-2.1` · `Nano-banana-2.1【VIP】` |
| Imagen | `GPT-image-2.5-sunburst【VIP】` · `GPT-image-2.5-flare【VIP】` · `GPT Image 2.0【VIP】` |
| Texto | `gpt-6.1-sol` · `gemini-3.8-flash` · `gemini3.1-flash` |

## Feishu Base · vincula la tarea a la fila del SKU

1. Crea campos para SKU, versión, imagen de referencia, instrucción, modelo, estado, ID de tarea y resultado. Si el archivo adjunto requiere iniciar sesión en Feishu, obtén mediante un servicio autorizado un enlace directo que Senzhi pueda leer y que siga vigente durante el procesamiento.
2. Cuando se apruebe una fila y el ID esté vacío, usa la acción «发送 HTTP 请求» (enviar una solicitud HTTP) para hacer el POST anterior. Combina el ID de la fila, el uso previsto y la versión para formar un `idempotency_key` estable. Guarda la clave en una automatización de acceso restringido o en tu servidor. Escribe el `request_id` en la fila correspondiente.
3. En otro flujo programado, selecciona las filas pendientes que ya tienen ID y envía un GET con ese ID en el parámetro de consulta `request_id`. Guarda `result.url` o `result.content` cuando finalice. Para conservar el archivo, descárgalo en la biblioteca de tu equipo.

## ComfyUI · conecta las imágenes y el texto a tus nodos

Descarga el complemento del espacio de trabajo y extráelo de modo que `requirements.txt` quede directamente dentro de `ComfyUI/custom_nodes/ComfyUI-NewAPI-Bridge/`. Instala las dependencias con el entorno de Python que utiliza ComfyUI, reinicia el programa y actualiza el navegador. Busca `森智AI` o `NewAPI` para encontrar los nodos de configuración, imagen, texto y vista previa de texto. Configura el servicio y la clave en el nodo de conexión. Antes de compartir el flujo, borra la clave y revisa el JSON exportado. [Complemento e instrucciones de instalación](https://www.senzhiai.com/workspace.php?tool=open_api)
