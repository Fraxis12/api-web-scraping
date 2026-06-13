# API de Web Scraping

Este proyecto obtiene información de una tabla pública del portal de los
Bomberos del Perú, transforma cada fila en un objeto y guarda el resultado en
una tabla de Amazon DynamoDB.

La solución está preparada para ejecutarse como una función de AWS Lambda y
exponerse mediante un endpoint HTTP usando Serverless Framework.

## Objetivo

Automatizar el siguiente proceso:

1. Consultar la página de emergencias de las últimas 24 horas.
2. Localizar la primera tabla disponible en el HTML.
3. Extraer sus encabezados y filas.
4. Eliminar los registros anteriores de DynamoDB.
5. Insertar la información recién obtenida.
6. Devolver los datos procesados como respuesta del endpoint.

## Tecnologías

- Python 3.12
- Requests para realizar la petición HTTP
- Beautiful Soup para analizar el HTML
- Boto3 para comunicarse con DynamoDB
- AWS Lambda para ejecutar el scraper
- Amazon API Gateway para publicar el endpoint
- Amazon DynamoDB para almacenar los resultados
- Serverless Framework para desplegar la infraestructura

## Estructura del proyecto

```text
.
├── README.md
├── requirements.txt
├── scrap_table.py
└── serverless.yml
```

### `scrap_table.py`

Contiene la función principal `lambda_handler(event, context)`. Esta función:

- Realiza una petición `GET` a:
  `https://sgonorte.bomberosperu.gob.pe/24horas/?criterio=/`
- Comprueba que el sitio responda correctamente.
- Analiza el HTML con Beautiful Soup.
- Busca la primera etiqueta `<table>`.
- Obtiene los encabezados desde las etiquetas `<th>`.
- Convierte las celdas `<td>` de cada fila en un diccionario.
- Se conecta a la tabla `TablaWebScrapping`.
- Elimina los elementos almacenados anteriormente.
- Agrega un número correlativo en el campo `#`.
- Genera un identificador UUID en el campo `id`.
- Inserta cada registro nuevo en DynamoDB.

### `requirements.txt`

Define las dependencias externas que deben incluirse en el despliegue:

```text
beautifulsoup4==4.9.3
requests==2.25.1
```

`boto3` no aparece en este archivo porque está disponible en el entorno de
ejecución administrado de AWS Lambda. Para ejecutar el proyecto localmente sí
puede ser necesario instalarlo.

### `serverless.yml`

Describe la infraestructura que Serverless Framework desplegará en AWS:

- Servicio: `api-web-scraping`
- Runtime: Python 3.12
- Memoria de Lambda: 1024 MB
- Tiempo máximo de ejecución: 30 segundos
- Función: `scrape_table`
- Handler: `scrap_table.lambda_handler`
- Método HTTP: `GET`
- Ruta: `/scrape/table`
- CORS habilitado
- Tabla DynamoDB: `TablaWebScrapping`
- Clave primaria de DynamoDB: `id`, de tipo texto
- Facturación de DynamoDB: bajo demanda (`PAY_PER_REQUEST`)

El archivo utiliza actualmente este rol de AWS:

```text
arn:aws:iam::979121676582:role/LabRole
```

Ese rol debe existir en la cuenta utilizada para el despliegue y tener permisos
para ejecutar Lambda, escribir en DynamoDB y trabajar con los demás recursos
administrados por Serverless Framework.

## Flujo de la aplicación

```text
Cliente
   |
   | GET /scrape/table
   v
API Gateway
   |
   v
AWS Lambda
   |
   | Solicitud HTTP
   v
Portal de Bomberos del Perú
   |
   | Tabla HTML
   v
Beautiful Soup
   |
   | Filas convertidas en objetos
   v
Amazon DynamoDB
   |
   v
Respuesta HTTP con los datos extraídos
```

## Requisitos

Para trabajar con el proyecto se necesita:

- Python 3.12
- `pip`
- Node.js y npm
- Serverless Framework
- Credenciales de AWS configuradas
- Acceso al portal utilizado como fuente de datos

## Instalación local

Crear y activar un entorno virtual:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Instalar las dependencias:

```bash
pip install -r requirements.txt
pip install boto3
```

Para ejecutar la función localmente también deben existir credenciales de AWS
válidas, ya que el código intenta leer y escribir directamente en DynamoDB.

## Ejecución local

La función puede invocarse desde Python:

```bash
python3 -c "from scrap_table import lambda_handler; print(lambda_handler({}, None))"
```

Esta ejecución no es solamente una prueba de lectura: elimina los elementos
actuales de `TablaWebScrapping` y después inserta los datos obtenidos.

También se puede invocar mediante Serverless Framework:

```bash
npx serverless invoke local --function scrape_table
```

## Despliegue

Instalar Serverless Framework si todavía no está disponible:

```bash
npm install --global serverless
```

Configurar las credenciales de AWS según el entorno y desplegar:

```bash
npx serverless deploy
```

Al terminar, Serverless mostrará la URL pública generada por API Gateway. El
endpoint tendrá una estructura similar a:

```text
GET https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/scrape/table
```

Para invocar directamente la función desplegada:

```bash
npx serverless invoke --function scrape_table
```

Para retirar los recursos creados:

```bash
npx serverless remove
```

## Respuestas

Cuando el scraping y el almacenamiento terminan correctamente:

```python
{
    "statusCode": 200,
    "body": [
        {
            "#": 1,
            "id": "uuid-generado",
            "...": "datos extraídos de la tabla"
        }
    ]
}
```

Si el sitio web responde con un error:

```python
{
    "statusCode": 500,
    "body": "Error al acceder a la página web"
}
```

El código conserva en `statusCode` el código HTTP recibido del sitio de origen,
por lo que no siempre será exactamente `500`.

Si no se encuentra una tabla:

```python
{
    "statusCode": 404,
    "body": "No se encontró la tabla en la página web"
}
```

## Consideraciones importantes

- Cada ejecución reemplaza los datos existentes en DynamoDB.
- El scraper depende de la estructura HTML del portal. Un cambio en la tabla,
  sus encabezados o sus columnas puede requerir cambios en el código.
- Actualmente se utiliza la primera tabla encontrada en la página.
- La petición HTTP no define un tiempo máximo de espera.
- El borrado usa una sola operación `scan`; DynamoDB pagina sus resultados
  cuando hay muchos datos, por lo que una tabla grande requeriría recorrer todas
  las páginas.
- La extracción usa `headers[i + 1]`, lo que supone que existe un encabezado
  adicional al inicio de la tabla.
- No hay manejo específico de excepciones de red, errores de HTML ni errores
  enviados por AWS.
- La función realiza una escritura individual por cada fila extraída.
- El rol de IAM está escrito directamente en `serverless.yml`; si cambia la
  cuenta o el entorno, debe actualizarse.

## Posibles mejoras

- Añadir un `timeout` a la petición HTTP.
- Validar que la cantidad de encabezados coincida con la cantidad de celdas.
- Manejar excepciones y registrar errores con CloudWatch.
- Procesar todas las páginas del `scan` de DynamoDB.
- Usar `batch_writer` también para las inserciones.
- Configurar la URL, el nombre de la tabla y el rol mediante variables de
  entorno.
- Agregar pruebas unitarias usando respuestas HTML simuladas.
- Programar el scraping con EventBridge para ejecutarlo automáticamente.
- Actualizar las dependencias y comprobar su compatibilidad periódicamente.

## Advertencia sobre los datos

La información pertenece al sitio web consultado. Antes de usarla, almacenarla
o redistribuirla, se deben revisar sus términos de uso, políticas y cualquier
restricción aplicable. También es recomendable evitar solicitudes demasiado
frecuentes para no afectar el servicio de origen.

## Preguntas para conversar con el profesor

1. ¿Por qué conviene ejecutar este scraper en AWS Lambda en lugar de mantener
   un servidor funcionando permanentemente?
2. ¿Qué sucedería si el sitio web cambia el orden o el nombre de las columnas
   de su tabla?
3. ¿Sería más seguro identificar la tabla por un `id` o una clase CSS en lugar
   de seleccionar la primera tabla de la página?
4. ¿Por qué el código utiliza `headers[i + 1]` y en qué casos podría producir
   un error?
5. ¿Qué estrategia podríamos utilizar para no eliminar toda la información de
   DynamoDB en cada ejecución?
6. ¿Sería mejor conservar un historial de emergencias o únicamente la
   información más reciente?
7. ¿Cómo podríamos detectar registros repetidos sin generar un UUID diferente
   en cada ejecución?
8. ¿Qué clave primaria sería más útil en DynamoDB si quisiéramos consultar las
   emergencias por fecha, ubicación o tipo?
9. ¿Qué ocurre si DynamoDB devuelve varias páginas de resultados durante el
   `scan`?
10. ¿Cuándo resulta más eficiente usar `batch_writer` en lugar de ejecutar
    `put_item` por cada registro?
11. ¿Qué excepciones deberíamos controlar si el sitio tarda demasiado, no tiene
    conexión o devuelve HTML incompleto?
12. ¿Cómo podríamos probar el scraper sin acceder al sitio real ni modificar
    la tabla de DynamoDB?
13. ¿Qué información sería útil registrar en CloudWatch para detectar errores
    y medir el funcionamiento de la Lambda?
14. ¿Qué permisos mínimos debería tener el rol de IAM para aplicar el principio
    de menor privilegio?
15. ¿Qué ventajas tendría guardar la URL y el nombre de la tabla en variables
    de entorno?
16. ¿Cómo podríamos ejecutar el scraping automáticamente cada cierto tiempo con
    Amazon EventBridge?
17. ¿Cómo evitaríamos realizar demasiadas solicitudes y afectar al sitio web de
    origen?
18. ¿Qué aspectos legales o éticos se deben revisar antes de extraer y publicar
    datos de una página web?
19. ¿En qué casos sería preferible consumir una API oficial en vez de hacer web
    scraping?
20. ¿Cómo cambiaría esta solución si la tabla fuera generada con JavaScript y
    no estuviera presente en el HTML inicial?
