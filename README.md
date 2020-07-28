# Trabajo de ASA Nica 2k19 NicaVentas

En este documento se describe la propuesta del trabajo a realizar como parte de las actividades de evalución del módulo de Administración de Servicios de Aplicación, impartido en el mes de julio de 2019.

Se trata de un caso de uso hipotético que requiere emplear las habilidades desarolladas durante el módulo, en concreto API REST, arquitecturas cloud, containerización con docker, programación de microservicios y el patrón de invalidación de cache que estudiamos en clase.

El desarrollo se ha dividido en varias etapas o niveles. Cada uno de estos niveles tiene una calificación asociada. El estudiante puede decirdir hasta qué nivel quiere desarrollar.


<!-- toc -->

- [Caso de uso](#Caso-de-uso)
- [Servicio de consulta de disponibilidad de ventas](#Servicio-de-consulta-de-disponibilidad-de-ventas)
- [Servicio de consulta de condiciones de venta](#Servicio-de-consulta-de-condiciones-de-venta)
- [API OpenWeather](#API-OpenWeather)
- [Fases de desarrollo](#Fases-de-desarrollo)
  * [Nivel 1](#Nivel-1)
    + [Objetivos:](#Objetivos)
    + [Entregables:](#Entregables)
    + [Pruebas a superar:](#Pruebas-a-superar)
  * [Nivel 2](#Nivel-2)
    + [Objetivos:](#Objetivos-1)
    + [Entregables:](#Entregables-1)
    + [Pruebas a superar:](#Pruebas-a-superar-1)
  * [Nivel 3](#Nivel-3)
    + [Objetivos:](#Objetivos-2)
    + [Entregables:](#Entregables-2)
    + [Pruebas a superar:](#Pruebas-a-superar-2)
  * [Nivel 4](#Nivel-4)
    + [Objetivos:](#Objetivos-3)
    + [Entregables:](#Entregables-3)
    + [Pruebas a superar:](#Pruebas-a-superar-3)

<!-- tocstop -->

## Caso de uso

Una empresa nos solicita el desarrollo de un API rest para que una serie de aplicaciones puedan consultar si pueden o no realizarse ventas en una determinada localidad de un país, y de ser posible, qué variaciones deben aplicar sobre el precio de los servicios en función de una serie de reglas. Con este fin desarrollaremos dos servicios: consulta de disponibilidad de ventas y servicio de consulta de condiciones de venta. A continuación se describen cada uno de estos servicios

## Servicio de consulta de disponibilidad de ventas

Servicio web se emplea para consultar si se está autorizada la venta de productos en general en una ciudad concreta de un país. Para ello se contruirá un API REST, y concretamente para esta consulta se implementará un endpoint `[GET] /active?city=leon&country=ni`.

El resultado de la invocación de este endpoint, a modo de ejemplo, será el siguiente:

```json
{
  "active": true,
  "country": "ni",
  "city": "Leon"
}
```

En este documento se especifican siempre los paises haciendo uso del estándar  [ISO 3166-1 alpha-2 - Wikipedia](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)


El campo `active` indica si la venta está autorizada (`true`) o no (`false`) en la correspondiente ciudad (`city`) del país (`country`) especificado en la llamada.

Una serie de operadores son los encargados de activar y desactivar las posibilidades de venta en las ciudades. Estos operadores el siguiente endpoint del API para activar o desactivar la venta:


Modificar el estado de actividad de una ciudad de un país:
**URL**: `/active`
**Method**: `PUT`
**Auth required**: YES
**Body format**: `Content-type: application/json`
**Body payload**: 
```
{
  "active": true,
  "country": "ni",
  "city": "Leon"
}
```


Esta llamada solo se atenderá si incluye en las cabeceras HTTP un token de autenticación como el siguiente:

`Authorization: Bearer 2234hj234h2kkjjh42kjj2b20asd6918`

El token es un secreto compartido entre los encargados y el sistema. Para este ejemplo, el token `2234hj234h2kkjjh42kjj2b20asd6918` será siempre este.

## Servicio de consulta de condiciones de venta

El servicio de condiciones de venta permite consultar qué porcentaje de descuento se hará a un producto determinado. Los productos se identifican mediante un código único denominado SKU. A modo de ejemplo vamos a considerar dos productos:

|   SKU   |          DESCRIPCION         | PRECIO |
|:-------:|:-----------------------------|-------:|
| AZ00001 | Paraguas de señora estampado |    10€ |
| AZ00002 | Helado de sabor fresa        |     1€ |

El precio final de venta dependerá de dos factores: la ciudad y país de venta, y la condiciones meteorológicas de esa ciudad. La idea general es vender más caros los paraguas y más baratos los helados si estuviera lloviendo, y al contrario, abaratar los paraguas y encarecer helados si hiciera sol. Se proporcionará para esta consulta el endpoint `[GET] /price/<:sku>`.

Por ejemplo, si la venta se hace en León (Nicaragua) y está lloviendo en ese momento, la llamada `[POST] /quote` con body:
```
{
  "sku": "AZ00001",
  "country": "ni",
  "city": "Leon"
}
```
Respondería, por ejemplo:

```
{
  "sku": "AZ00001",
  "description": "Paraguas de señora estampado",
  "country": "ni",
  "city": "Leon",
  "base_price": 10,
  "variation": 1.5
}
```

El precio de los paraguas bajo estas condiciones sería de `10 x 1.5 = 15€`. 

Para calcular la respuesta adecuada, el endpoint `[POST] /quote` dispondrá del API de un tercero, concretamente de OpenWeather, para consultar el tiempo meteorológico de una ciudad concreta de un país. El uso de este API se describe más adelante, en [API OpenWeather](#API-OpenWeather). Se puede preguntar por las condiciones meteorológicas de una ciudad y país haciendo uso de la siguiente llamada:

```curl
curl "api.openweathermap.org/data/2.5/weather?q=Chinandega,ni&appid=3225ae99d4c4cb46be4a2be004226918"
```

O lo que es lo mismo, haciendo una petición `[GET]` a api.openweathermap.org/data/2.5/weather?q=Chinandega,ni&appid=3225ae99d4c4cb46be4a2be004226918

Esta petición informa de muchos datos sobre las condiciones meteorológicas en León, Nicaragua. Nos quedaremos con la parte de `weather`, y concretamente con el `id`, que nos indica las condiciones meteorológicas. Si consultamos la página del proveedor del API, concretemente en [Weather Conditions - OpenWeatherMap](https://openweathermap.org/weather-conditions), veremos que la condición de lluvia es el grupo 5XX, es decir, todos aquellos `id` comprendidos entre 500 y 599. Con esta información ya estamos en disposición de crear una tabla de reglas en la BBDD:


| id_regla | ciudad | pais |     SKU | min_condition | max_condition | variation |
|---------:|-------:|-----:|--------:|--------------:|--------------:|----------:|
|        1 |   Leon |   NI | AZ00001 |           500 |           599 |       1.5 |
|        2 |   Leon |   NI | AZ00002 |           500 |           599 |       0.5 |
|          |        |      |         |               |               |           |

Supongamos que preguntamos al servicio meteorológico sobre las condiciones en Leon, Nicaragua, y obtenemos id=503 (very heavy rain). Consultamos a continuación a la base de datos:

```sql
SELECT MAX(variation) 
  FROM rules 
  WHERE
    city = "Leon" AND
    country = "ni" AND
    min_condition >= weather_id AND
    max_condition <= weather_id AND
    SKU = "AZ0001"
```

Si se cumple al menos una regla obtendremos un valor en esta consulta, y entonces ese valor será la variación que debemos usar. Si por el contrario no se cumpliera ninguna regla, la consulta no devolvería ningún valor y se podría considerar que la variación es 1, o lo que es lo mismo, que no hay variación.

Para obtener la descripción y el precio base crearemos otra tabla:

| SKU     | Description                  |
|---------|------------------------------|
| AZ00001 | Paraguas de señora estampado |
| AZ00002 | Helado de sabor fresa        |
|         |                              |


## API OpenWeather

Utilizar el API es muy sencillo. Solo hay que registrarse con una dirección de email y solicitar a continuación un API key para su oferta de servicios gratuitos. Una vez obtenido este API hay que usarlo en todas las llamadas, pasandolo como un `query_param` en la URL. Por ejemplo, si el API fuera 12345678901234567890 la llamada a realizar para obtener las condiciones en Chinandega, Nicaragua sería la siguiente:

```curl
curl "api.openweathermap.org/data/2.5/weather?q=Chinandega,ni&appid=12345678901234567890"
```

Si consultamos https://openweathermap.org/weather-conditions podemos obtener la descripción de uno de los campos informativos, el campo `id` de `weather`. Hay que tener en cuenta que el apartado `weather` es una lista, y por lo tanto puede haber más de un `id`.


## Fases de desarrollo
Este ejercicio se puede completar de forma incremental. Para ello se han dividido los objetivos en varios niveles. Cada nivel tiene asociada una calificación sujeta al cumplimiento de una serie de objetivos que se detallan a continuación

### Nivel 1
#### Objetivos:
  - Desarrollar un microservicio en `flask` que implemente la llamada `[GET] /active` con una respuesta _dummy_ fija
  - Crear una imagen docker que contenga dicho microservicio y publicarla en `dockerhub`

#### Entregables:
- Documento explicativo del trabajo realizado 
- URL de dockerhub para descargar la imagen, p.e [(https://hub.docker.com/r/opobla/nica-ventas)](https://hub.docker.com/r/opobla/nicatime-locust)
- Archivos necesarios para construir la imagen docker (archivo `Dockerfile` y archivos de código fuente necesarios.)

#### Pruebas a superar:

Arrancar el contenedor haciendo:
```bash
docker pull opobla/nica-ventas
docker run -p 8000 opobla/nica-ventas
```

Probar con `curl localhost:8000/active?city=leon&country=ni`. La respuesta debe ser una respuesta JSON válida conforme a [la especificación del servicio](#Servicio-de-consulta-de-disponibilidad-de-ventas)

:moneybag:Calificación 20% 

### Nivel 2

#### Objetivos:
  - Ampliar el microservicio par que implemente la llamada `[POST] /active`. El estado, la ciudad y el país se deberá almacenar en una base de datos relacional.
  - Modificar el microservicio para que la llamada `[GET] /active` obtenga sus resultados desde la base de datos.
  - Orquestar el funcionamiento del microservicio con el de la base de datos haciendo uso de `docker-compose`. La base de datos en concreto es indiferente, pero se recomienda utilizar `postgres`, `mysql` o `mariadb`
  - Crear una imagen docker que contenga dicho microservicio y publicarla en `dockerhub`

#### Entregables:
- Documento explicativo del trabajo realizado 
- Archivo `docker-compose`

#### Pruebas a superar:

Arrancar los servicios haciendo:
```bash
docker-compose up
```

Pruebas con curl haciendo un POST a una ciudad de un pais arbitrario y acto seguido obteniendo los valores con el GET correspondiente

:moneybag:Calificación 40%

### Nivel 3

#### Objetivos:
  - Modificar el microservicio para que la respuesta de la llamada `[GET] /active` se haga desde una cache REDIS. La respuesta de este servicio ahora incluirá un campo `cache` que valdrá `miss` si la respuesta procede de la base de datos, y `hit` si la respuesta procede de caché.
  - Modificar el microservicio para que la respuesta de la llamada `[POST] /active` invalide el posible contenido cacheado en REDIS.
  - Añadir al archivo `docker-compose` el servicio de `redis`

#### Entregables:
- Documento explicativo del trabajo realizado 
- Archivo `docker-compose`

#### Pruebas a superar:

1) Arrancar los servicios haciendo:
```bash
docker-compose up
```
2) Hacer una petición POST con una ciudad, pais y estado concretos.
3) Hacer una petición GET para la misma ciudad y pais. Verificar que para esta primera petición el campo `cache` deber ser `miss`.
4) Hacer una segunda petición GET para la misma ciudad y pais. Verificar que para esta segunda petición y las consecutivas el campo `cache` deber ser `hit`.

:moneybag: Calificación 60%

### Nivel 4

#### Objetivos:
  - Evolucionar la arquitectura existente para incluir un micro servicio que proporcione el API `[POST] /quote`, según lo especificado en el enunciado en [Servicio de consulta de condiciones de venta](Servicio-de-consulta-de-condiciones-de-venta)
  - Añadir al archivo `docker-compose` el nuevo microservicio
  - Añadir políticas de cacheo de forma que si se solicita `[POST] /quote` con los mismos parámetros se responda desde la cache de REDIS en lugar de volver a realizar la consultas a OpenWeather y la BBDD. La valided de uno de estos datos cacheados será de 5 min. Con objeto de verificar que la cache funciona, incluir en la respuesta un campo `cache` como se hizo anteriormente.

#### Entregables:
- Documento explicativo del trabajo realizado 
- Archivo `docker-compose`
- Script para hacer la carga inicial de los valores iniciales de las tablas de la base de datos.
- 
#### Pruebas a superar:

1) Arrancar los servicios haciendo:
```bash
docker-compose up
```

Hacer pruebas específicas con distintas llamadas para comprobar que se disparan las reglas apropiadas en casa caso y que se cachean las respuestas.

:moneybag: Calificación 100%
