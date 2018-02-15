# Sistema de telefonía de prepago


## Uso del API para módulo de telefonía de prepago

### Fase "Usuario CÓDIGO":

Se corresponde con el instante (3) descrito en el cronograma.

##### Verificación de propio número de teléfono

```http
GET /phones/?number=961992899 HTTP/1.1
```

Se obtiene la información sobre la **disponibilidad del propio número de teléfono** (comprobación de validez). En el ejemplo se verifica la existencia de al menos un teléfono registrado con el número 961992899. 

Respuesta: *OK*
```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data":  [
    {
      "id": "7",
      "number": "961992899",
      // resto de campos
    }
  ]
}
```

Respuesta: *Número no registrado*
```http
HTTP/1.1 404 Not found
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "404",
      "source": { "pointer": "/data/number" },
      "code": 101,
      "title": "Not registered number",
      "detail": "The phone is unreachable because it has not any number associated"
    }
 ]
}
```

---

##### Saldo disponible

```http
GET /payment-objects/?ticket=12345 HTTP/1.1
```

Se obtiene la información acerca del **saldo disponible** consultando por el número de tiquet (código reutilizable). En el ejemplo, se piden los datos relacionados con el número de tiquet introducido por teléfono, en este caso, 12345. 

De la información obtenida se debería guardar tanto con el **identificador único de tiquet**, `id` como con el **saldo**, `balance`, para ser usados en nuevas consultas al API.


Respuesta: *OK*
```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data": [
    {
      "id": "2398012",
      "ticket": 12345,
      "balance": 12.40,
      "locks": [],
      // resto de campos
    }
  ]
}
```

Respuesta: *Tiquet no encontrado*

```http
HTTP/1.1 404 Not found
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "404",
      "source": { "pointer": "/data/ticket" },
      "code": 201,
      "title": "Ticket not found",
      "detail": "Cannot find any valid object with this ticket number"
    }
 ]
}
```

Respuesta: *Tiquet sin saldo*

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "400",
      "source": { "pointer": "/data/balance" },
      "code": 202,
      "title": "Exhausted balance",
      "detail": "The payment object has an exhausted balance"
    }
 ]
}
```

---

### Fase "Usuario DESTINO":

Se corresponde con el instante (6) descrito en el cronograma.

##### Tarifa de telefonía

```http
GET /phone-rates/?number_match=962331295&enabled=true HTTP/1.1
```

Se obtiene la información relacionada con la **tarifa** a aplicar para el número marcado.

De la información obtenida se deberían guardar los campos relacionados con los **costes de establecimiento** , `setup_fare` y el **coste por minuto**, `per_minute_fare`.


Respuesta: *Ok*

```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data": [
    {
      "id": "3",
      "name": "Locales",
      "description": "Tarificación local",
      "setup_fare": 0.18,
      "per_minute_fare": 0.05,
      "rule": "^96[0-9]{7}$",
      "sequence": 1,
      "enabled": true,
      "published": true
    }
  ]
}
```

Respuesta: *Ninguna tarifa encaja con el número de teléfono*

```http
HTTP/1.1 404 Not found
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "404",
      "source": { "pointer": "/data/number" },
      "code": 203,
      "title": "No phone number match",
      "detail": "Couldn't match any phone rate with the phone number"
    }
 ]
}
```

---

### Fase "Gestión llamada":

Se corresponde con el instante (8) descrito en el cronograma.

##### Creación de bloqueo de tiquet

```http
POST /payment-objects/2398012/locks/ HTTP/1.1
```
```javascript
{ 
  "data": {
    "reason": "unconsolidated-balance"
  }
}
```
Se crea un nuevo **bloqueo de tiquet** frente a nuevas compras, a fin de evitar nuevas compras concurrentes antes de consolidarse el saldo definitivo.

Se usa en este caso en el API el identificador de tiquet que se guardado previamente.

Respuesta: *Ok*

```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data": {
      "id": "101219",
      "reason": "unconsolidated-balance",
      "time": "2018-01-06T11:57:50Z",
      "expiration_time": "2018-01-06T23:57:50Z",
      // resto de campos
  }
}
```

Respuesta: *Tiquet ya está bloqueado*

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "400",
      "source": { "pointer": "/data/reason" },
      "code": 202,
      "title": "Ticket already locked because of unconsolidated balance",
      "detail": "The ticket is already locked and cannot be locked again because of unconsolidated balance"
    }
 ]
}

```

--- 

### Fase "Fin llamada":

Se corresponde con el instante (9) descrito en el cronograma.

##### Consolidación de saldo

```http
POST /payment-objects/2398012/operations/ HTTP/1.1
```
```javascript
{ 
  "data": {
    "type": "charge"
    "amount": 12.20
  }
}
```

Se produce el **cargo del coste de la llamada** sobre el tiquet, consolidándose el consumo. Simultáneamente se elimina el bloqueo de tiquet relacionado con consumos no consolidados.

Se usa en este caso en el API el identificador de tiquet guardado.


Respuesta: *Ok*

```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data": {
      "id": "8293121",
      "type": "charge",
      "amount": 12.20,
      "time": "2018-01-06T11:57:50Z",
      ...
  }
}
```

Respuesta: *Imposible consolidar el saldo*

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "400",
      "source": { "pointer": "/data/balance" },
      "code": 202,
      "title": "Couldn't consolidate balance in ticket",
      "detail": "..."
    }
 ]
}
```

---

##### Registro de llamada

```http
POST /phone-calls/ HTTP/1.1
```
```javascript
{
  "data": {
    "time": "2018-01-06T11:57:50Z",
    "src_number": "961992899",
    "dst_number": "962331295",
    "duration": 2123,
    "billed_duration": 2110,
    "phone_rate": 3,
    "amount": 2.6,
    "status": "answered"
  }
}
```
Se registran todos los los datos relacionados con la llamada.

Respuesta: *Ok*

```http
HTTP/1.1 200 OK
Content-Type: application/json
```
```javascript
{
  "data": {
      "time": "2018-01-06T11:57:50Z",
      "phone": "3",
      "src_number": "961992899",
      "dst_number": "962331295",
      "duration": 2123,
      "billed_duration": 2110,
      "phone_rate": 3,
      "amount": 3.0,
      "status": "answered"
  }
}
```

Respuesta: *Imposible crear la entrada en el registro*

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```
```javascript
{
  "errors": [
    {
      "status": "400",
      "source": { "pointer": "/data/balance" },
      "code": 202,
      "title": "Couldn't register call",
      "detail": "..."
    }
 ]
}
```


## Autenticación

Para la autenticación se usará HTTP Basic Authentication, proporcionándose las credenciales de acceso.
