# Análisis y Procesamiento de Datos de Twitter (X) en MongoDB

Este proyecto contiene un flujo de trabajo desarrollado en Python dentro de un entorno Jupyter Notebook (Google Colab) para la conexión, extracción, agregación y actualización de métricas de cuentas de redes sociales almacenadas en una base de datos NoSQL (MongoDB Atlas).

## Descripción del Proyecto

El objetivo principal del script es consolidar información de publicaciones desde una colección de tweets (`samples`) mediante el framework de agregación de MongoDB y actualizar los perfiles correspondientes en una colección de cuentas (`tweetsAccount`) dentro de la base de datos `Twitter`. Adicionalmente, se documentan las adaptaciones requeridas para integrar la ingesta directa en tiempo real desde la API oficial de X (anteriormente Twitter).

### Funcionalidades Principales

* **Conexión Segura:** Implementación de `getpass` para solicitar e introducir la URI de conexión a MongoDB Atlas (y tokens de API) de forma oculta en tiempo de ejecución, evitando exponer credenciales en el código fuente.
* **Agregación de Datos:** Uso de pipelines de MongoDB (`$group` y `$addFields`) para agrupar publicaciones por nombre de usuario (`user.screen_name`) y calcular el total acumulado de estados (`statuses_count`) y amigos (`friends_count`).
* **Actualización de Colecciones:** Iteración sobre los resultados agregados mediante PyMongo para sincronizar y actualizar los documentos de cada cuenta (`update_one`).

## Requisitos Previos

Para ejecutar este proyecto necesitas:

* Python 3.8 o superior.
* Una instancia activa de MongoDB (local o en la nube mediante MongoDB Atlas).
* Las colecciones `tweetsAccount` y `samples` configuradas en la base de datos `Twitter`.
* (Opcional) Una cuenta de desarrollador en el Portal de Desarrolladores de X (Twitter API v2) si se desea realizar la ingesta directa desde la API.

## Instalación

1. Clona este repositorio en tu máquina local o ábrelo directamente en Google Colab:
   ```bash
   git clone https://github.com/tu-usuario/tu-repositorio.git
   cd tu-repositorio
   ```

2. Instala las dependencias necesarias ejecutando el siguiente comando:
   ```bash
   pip install pymongo pandas tweepy
   ```
   *(Nota: Se recomienda `tweepy` en sustitución de `twython` para compatibilidad con la API v2 actual).*

## Uso

1. Abre el archivo `.ipynb` en Jupyter Notebook, JupyterLab o Google Colab.
2. Ejecuta la celda de instalación de dependencias si aún no tienes los paquetes en tu entorno.
3. Ejecuta la celda de configuración e introduce tu cadena de conexión (URI) de MongoDB Atlas cuando el sistema lo solicite.
4. Ejecuta las celdas de agregación y actualización para procesar los registros y volcar las métricas en la colección `tweetsAccount`.

## Estructura de Datos

El script interactúa con dos colecciones principales dentro de la base de datos `Twitter`:

* **`samples` (Origen):** Contiene los documentos de los tweets, de donde se extraen los campos anidados `user.screen_name`, `user.statuses_count` y `user.friends_count`.
* **`tweetsAccount` (Destino):** Almacena la información maestra de los perfiles identificados por `Twitter_handle`.

## Adaptaciones para Consumo Directo desde la API de Twitter (X)

El notebook original opera sobre un conjunto de datos previamente volcado en MongoDB e importa la librería `twython` (diseñada para la obsoleta API v1.1). Si se desea modificar el flujo para consumir datos directamente desde la API actual (X API v2), se deben realizar los siguientes cambios en el código:

### 1. Actualización de Librerías
Sustituir `twython` por `tweepy` (o `requests`) para interactuar con los endpoints de la API v2:
```python
import tweepy
```

### 2. Autenticación mediante Bearer Token
En la sección de configuración de credenciales, añadir la solicitud segura del `Bearer Token` obtenido en el portal de desarrolladores:
```python
bearer_token = getpass("Pega tu Bearer Token de la API de X: ").strip()
client_api = tweepy.Client(bearer_token=bearer_token)
```

### 3. Ingesta de Datos hacia MongoDB (`samples`)
Antes de ejecutar el pipeline de agregación, se debe crear un bloque de extracción que consulte la API, transforme la respuesta JSON al esquema esperado por la colección y la inserte en MongoDB:
```python
# Ejemplo de obtención de métricas de usuario en API v2
handles = ["elmundoes", "el_pais", "RafaelNadal"]
users_data = client_api.get_users(usernames=handles, user_fields=["public_metrics"])

for user in users_data.data:
    metrics = user.public_metrics
    documento_tweet = {
        "user": {
            "screen_name": user.username,
            "statuses_count": metrics["tweet_count"],
            "friends_count": metrics["following_count"]
        }
    }
    tweets.insert_one(documento_tweet)
```

### 4. Ajuste en la Lógica de Agregación
Al consultar métricas directamente desde el perfil de usuario mediante la API v2, los valores devueltos (`tweet_count` y `following_count`) ya representan el total histórico de la cuenta y no un valor por cada tweet individual. Por lo tanto, en la etapa `$group` del pipeline de MongoDB se debe sustituir el operador `$sum` por `$last` o `$max` para evitar multiplicar los totales por la cantidad de documentos insertados:
```python
'$group': {
    '_id': '$user.screen_name',
    'c_total_tweets': {'$max': '$user.statuses_count'},
    'c_friends': {'$max': '$user.friends_count'}
}
```

## Notas Técnicas y Correcciones del Código Base

* **Claves duplicadas en actualización:** En el código actual (`accounts.update_one`), se utilizan dos claves `$set` consecutivas dentro del mismo diccionario de Python. En Python, las claves duplicadas se sobrescriben, por lo que únicamente se ejecutará la segunda instrucción (`num_amigos`). Para actualizar ambos campos simultáneamente, deben agruparse bajo un mismo operador `$set`:
  ```python
  accounts.update_one(
      {'Twitter_handle': org['_id']},
      {
          '$set': {
              'num_tweets_enviados': org['new_total_tweets'],
              'num_amigos': org['new_friends']
          }
      }
  )
  ```
* **Sintaxis de operadores de actualización:** El uso de `$push` y `$each` dentro de un operador `$set` provocará un error de sintaxis en MongoDB si el valor de entrada no es un arreglo (array) o si se intenta anidar un operador de actualización dentro de otro. Si el objetivo es almacenar un valor numérico entero, debe asignarse directamente dentro de `$set` como se muestra en el bloque anterior.
