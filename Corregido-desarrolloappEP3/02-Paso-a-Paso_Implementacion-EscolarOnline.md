# Anexo: Guia Paso a Paso — Implementacion Serverless Hibrido

## ARY1102 — Actividad 3.3 | EscolarOnline

| Campo | Valor |
|-------|-------|
| Caso | EscolarOnline (evolucion serverless) |
| Region | us-east-1 |
| Prerequisito | EKS + RDS de EA2 funcionando |

---

## Paso 1: Agregar tabla ordenes a RDS MySQL

Conectar a RDS MySQL y ejecutar init.sql


Verificar: `SHOW TABLES;` debe mostrar `productos` y `ordenes`.

---

## Paso 2: Crear tabla DynamoDB (carrito)

1. Ir a **DynamoDB** → **Create table**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Table name | carrito-escolaronline |
| Partition key | userId (String) |
| Sort key | productId (String) |
| Table settings | Default settings |
| Capacity mode | On-demand |

3. Despues de crear, habilitar TTL:
   - Ir a la tabla → **Additional settings** → **Time to Live (TTL)**
   - TTL attribute: `expiresAt`
   - Click **Turn on**

---

## Paso 3: Crear cola SQS

### Cola principal

1. Ir a **SQS** → **Create queue**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Type | Standard |
| Name | ordenes-escolaronline |
| Visibility timeout | 60 seconds |
| Message retention | 4 days |
| Delivery delay | 0 seconds |

### Dead Letter Queue (DLQ)

1. Crear otra cola:

| Campo | Valor |
|-------|-------|
| Type | Standard |
| Name | ordenes-escolaronline-dlq |

2. Volver a la cola principal → **Edit** → **Dead-letter queue**:
   - Enabled: Yes
   - Queue: ordenes-escolaronline-dlq
   - Maximum receives: 3

3. **Anotar la Queue URL** (ej: `https://sqs.us-east-1.amazonaws.com/123456789012/ordenes-escolaronline`)

---

## Paso 4: Crear VPC Endpoint Gateway para DynamoDB

1. Ir a **VPC** → **Endpoints** → **Create endpoint**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Name | escolaronline-dynamodb-endpoint |
| Service category | AWS services |
| Service | com.amazonaws.us-east-1.dynamodb |
| VPC | escolaronline-vpc |
| Route tables | Seleccionar las route tables de subredes privadas APP |

3. Click **Create endpoint**

Esto permite que Lambda (en VPC) acceda a DynamoDB sin salir a internet.

---

## Paso 5: Crear IAM Roles para Lambda

### Role: LambdaCartRole

Permisos:
- AWSLambdaBasicExecutionRole (logs)
- DynamoDB: PutItem, GetItem, DeleteItem, Query en tabla `carrito-escolaronline`

Policy JSON:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:GetItem",
                "dynamodb:DeleteItem",
                "dynamodb:Query"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:*:table/carrito-escolaronline"
        }
    ]
}
```

### Role: Verificar role en AWS Academy (LabRole ó LabInstanceProfile)

Permisos:
- AWSLambdaBasicExecutionRole
- DynamoDB: Query en tabla `carrito-escolaronline`
- SQS: SendMessage en cola `ordenes-escolaronline`

### Role: Verificar role en AWS Academy (LabRole ó LabInstanceProfile)

Permisos:
- AWSLambdaBasicExecutionRole
- AWSLambdaVPCAccessExecutionRole (para acceder a RDS en VPC)
- SQS: ReceiveMessage, DeleteMessage, GetQueueAttributes
- DynamoDB: Query, BatchWriteItem en tabla `carrito-escolaronline`

> Nota: En AWS Academy, usar `LabRole` si no se pueden crear roles custom. Verificar role en AWS Academy (LabRole ó LabInstanceProfile)

---

## Paso 6: Crear Lambda Cart (procesar-carrito)

1. Ir a **Lambda** → **Create function**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Function name | escolaronline-procesar-carrito |
| Runtime | Node.js 18.x |
| Architecture | arm64 |
| Execution role | LambdaCartRole (o LabRole) |

3. En **Code source**: pegar el contenido de `lambda/procesar-carrito/index.js`
4. En **Configuration** → **Environment variables**:
   - `TABLE_NAME` = `carrito-escolaronline`
5. En **Configuration** → **General configuration**:
   - Memory: 128 MB
   - Timeout: 30 seconds

6. Click **Deploy**

### Probar Lambda Cart

Test event (agregar al carrito):
```json
{
    "httpMethod": "POST",
    "body": "{\"userId\":\"test-user\",\"productId\":\"1\",\"nombre\":\"Cuaderno\",\"precio\":2990,\"cantidad\":2}",
    "queryStringParameters": null,
    "pathParameters": null
}
```

Test event (consultar carrito):
```json
{
    "httpMethod": "GET",
    "body": null,
    "queryStringParameters": {"userId": "test-user"},
    "pathParameters": null
}
```

---

## Paso 7: Crear Lambda Create-Order (crear-orden)

1. Ir a **Lambda** → **Create function**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Function name | escolaronline-crear-orden |
| Runtime | Node.js 18.x |
| Architecture | arm64 |
| Execution role | LambdaCreateOrderRole (o LabRole) |

3. Pegar codigo de `lambda/crear-orden/index.js`
4. Environment variables:
   - `TABLE_NAME` = `carrito-escolaronline`
   - `QUEUE_URL` = `https://sqs.us-east-1.amazonaws.com/<ACCOUNT_ID>/ordenes-escolaronline`
5. Memory: 128 MB, Timeout: 30 seconds
6. Deploy

### Probar

```json
{
    "httpMethod": "POST",
    "body": "{\"userId\":\"test-user\"}"
}
```

Resultado esperado: mensaje "Su pedido fue recibido" + verificar mensaje en cola SQS.

---

## Paso 8: Crear Lambda Process-Order (procesar-orden)

1. Ir a **Lambda** → **Create function**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Function name | escolaronline-procesar-orden |
| Runtime | Node.js 18.x |
| Architecture | arm64 |
| Execution role | LambdaProcessOrderRole (o LabRole) |

3. Pegar codigo de `lambda/procesar-orden/index.js`
4. Environment variables:
   - `DB_HOST` = `<RDS_ENDPOINT>`
   - `DB_USER` = `alumno`
   - `DB_PASS` = `alumno123`
   - `DB_NAME` = `escolar_online`
   - `DB_PORT` = `3306`
   - `TABLE_NAME` = `carrito-escolaronline`
5. Memory: 256 MB, Timeout: 60 seconds
6. **VPC Configuration** (necesario para acceder a RDS):
   - VPC: escolaronline-vpc
   - Subnets: subredes privadas APP (AZ1a + AZ1b)
   - Security Group: SG-EKS (o crear SG-Lambda con acceso a RDS, se puede crear un SG Lambda que tenga como unica regla la de salida para todos lados y luego agregar en el SG del RDS como regla de entrada la conexion desde el SG de Lambda)
7. Añadir mysql2 layer para que el codigo mande la orden generada a mysql<br>
Primero crear el layer/capa de mysql (correr en aws cloudshell directamente)
```
mkdir -p nodejs && cd nodejs && npm install mysql2 && cd .. && zip -r layer.zip nodejs && aws lambda publish-layer-version --layer-name mysql2 --zip-file fileb://layer.zip
```
8. Añadir esta capa personalizada (custom layer) en la funcion lambda<br>
Ir a la seccion Codigo (Code) <br>
Ir a la parte final de la pagina <br> 
Presionar Editar (Edit) <br>
Agregar Capa (Add Layer) <br>
Seleccionar Capa Personalizada (Custom Layer) y en el listado "mysql2" <br>
Guardar y Salvar

9. Deploy

### Configurar trigger SQS

1. En la funcion → **Add trigger**
2. Source: SQS
3. SQS queue: ordenes-escolaronline
4. Batch size: 1
5. Click **Add**

### Probar

Enviar un mensaje manualmente a la cola SQS y verificar que Lambda lo procesa y guarda en MySQL.

---

## Paso 9: Crear bucket S3 Static Website

1. Ir a **S3** → **Create bucket**
2. Configurar:

| Campo | Valor |
|-------|-------|
| Bucket name | escolaronline-frontend-<ACCOUNT_ID> |
| Region | us-east-1 |
| Block public access | Desmarcar "Block all public access" |
| Acknowledge | Marcar checkbox |

3. Despues de crear → **Properties** → **Static website hosting**:
   - Enable
   - Index document: `index.html`

4. **En la seccion Permissions (despues de terminar el paso anterior)** → **Bucket policy**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::escolaronline-frontend-<ACCOUNT_ID>/*"
        }
    ]
}
```

---

## Paso 10: Subir frontend a S3

Subir los archivos de `desarrolloApp-ACT3.3/S3Frontend/` al bucket:

```bash
aws s3 sync ./S3Frontend/ s3://escolaronline-frontend-<ACCOUNT_ID>/ --region us-east-1
```

O subir manualmente via consola S3 (Upload).

**Importante:** Antes de subir, actualizar `js/app.js` con la URL del API Gateway (se obtiene en paso 13).

---

## Paso 11: Crear API Gateway REST

1. Ir a **API Gateway** → **Create API** → **REST API** → **Build**
2. Configurar:
   - API name: `escolaronline-api`
   - Endpoint type: Regional

### Crear recursos y metodos:

| Resource | Method | Integration | Backend |
|----------|--------|-------------|---------|
| /api/products | GET | HTTP Proxy | http://EKS-ALB-DNS-URL/api/products |
| /api/products | POST | HTTP Proxy | http://EKS-ALB-DNS-URL/api/products |
| /api/products/{id} | PUT | HTTP Proxy | http://EKS-ALB-DNS-URL/api/products/{id} |
| /api/products/{id} | DELETE | HTTP Proxy | http://EKS-ALB-DNS-URL/api/products/{id} |
| /api/cart | GET | Lambda Proxy | escolaronline-procesar-carrito |
| /api/cart | POST | Lambda Proxy | escolaronline-procesar-carrito |
| /api/cart/{productId} | DELETE | Lambda Proxy | escolaronline-procesar-carrito |
| /api/orders | POST | Lambda Proxy | escolaronline-crear-orden |

### Habilitar CORS en cada resource:

- Seleccionar resource → **Actions** → **Enable CORS**
- Access-Control-Allow-Origin: `*`
- Access-Control-Allow-Methods: `GET, POST, PUT, DELETE, OPTIONS`

---

## Paso 12: Deploy API Gateway

1. **Actions** → **Deploy API**
2. Stage: **New Stage** → name: `prod`
3. Click **Deploy**
4. **Anotar la Invoke URL** (ej: `https://abc123.execute-api.us-east-1.amazonaws.com/prod`)

---

## Paso 13: Actualizar frontend con URL de API Gateway

Editar `frontend/js/app.js` y actualizar la variable `API_URL`:

```javascript
const API_URL = "https://abc123.execute-api.us-east-1.amazonaws.com/prod";
```

Re-subir a S3:
```bash
aws s3 cp ./frontend/js/app.js s3://escolaronline-frontend-<ACCOUNT_ID>/js/app.js
```

---

## Paso 14: Validacion Final

### URLs de acceso

| Servicio | URL |
|----------|-----|
| Frontend (S3) | http://escolaronline-frontend-<ACCOUNT_ID>.s3-website-us-east-1.amazonaws.com |
| API Gateway | https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod |

### Pruebas CRUD Productos (API GW → EKS)

```bash
API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod"

curl $API/api/products
curl -X POST $API/api/products -H "Content-Type: application/json" \
  -d '{"nombre":"Goma borrar","descripcion":"Goma blanca","precio":590,"stock":100,"categoria":"Escritura"}'
```

### Pruebas Carrito (API GW → Lambda → DynamoDB)

```bash
# Agregar al carrito
curl -X POST $API/api/cart -H "Content-Type: application/json" \
  -d '{"userId":"user001","productId":"1","nombre":"Cuaderno","precio":2990,"cantidad":2}'

# Consultar carrito
curl "$API/api/cart?userId=user001"

# Quitar del carrito
curl -X DELETE "$API/api/cart/1?userId=user001"
```

### Pruebas Ordenes (API GW → Lambda → SQS → Lambda → MySQL)

```bash
# Primero agregar items al carrito
curl -X POST $API/api/cart -H "Content-Type: application/json" \
  -d '{"userId":"user001","productId":"1","nombre":"Cuaderno","precio":2990,"cantidad":2}'
curl -X POST $API/api/cart -H "Content-Type: application/json" \
  -d '{"userId":"user001","productId":"3","nombre":"Lapices","precio":4990,"cantidad":1}'

# Crear orden
curl -X POST $API/api/orders -H "Content-Type: application/json" \
  -d '{"userId":"user001"}'

# Respuesta: {"message":"Su pedido fue recibido","orderId":"ORD-...","total":10970}

# Verificar en MySQL (esperar ~5 segundos)
# SELECT * FROM escolar_online.ordenes ORDER BY id DESC LIMIT 1;

# Verificar carrito limpio
curl "$API/api/cart?userId=user001"
# Respuesta: {"items":[],"total":0,"cantidadItems":0}
```

### Checklist final

- [ ] S3 Static Website accesible (frontend carga)
- [ ] DynamoDB tabla carrito-escolaronline activa con TTL
- [ ] SQS cola ordenes-escolaronline activa con DLQ
- [ ] VPC Endpoint Gateway para DynamoDB creado
- [ ] Lambda Cart funcional (POST, GET, DELETE)
- [ ] Lambda Create-Order funcional (publica en SQS)
- [ ] Lambda Process-Order funcional (trigger SQS → MySQL)
- [ ] API Gateway con 8 rutas desplegado
- [ ] CRUD productos funcional via API GW → EKS
- [ ] Carrito funcional via API GW → Lambda → DynamoDB
- [ ] Orden completa: crear → SQS → procesar → MySQL
- [ ] Carrito se limpia despues de confirmar orden
- [ ] Frontend end-to-end funcional

---

## Troubleshooting

| Problema | Causa probable | Solucion |
|----------|---------------|----------|
| Lambda Cart timeout | No puede acceder a DynamoDB | Verificar VPC Endpoint Gateway creado |
| Lambda Process-Order timeout | No conecta a RDS | Verificar VPC config + SG permite 3306 |
| API Gateway 403 | CORS no habilitado | Habilitar CORS en cada resource |
| API Gateway 502 | Lambda retorna formato incorrecto | Verificar que retorna statusCode + body + headers |
| SQS mensaje no se procesa | Trigger Lambda no configurado | Verificar trigger SQS en Lambda Process-Order |
| Frontend no carga | Bucket policy no permite acceso publico | Verificar bucket policy con s3:GetObject |
| Carrito no expira | TTL no habilitado | Verificar TTL attribute = expiresAt en DynamoDB |

---

## Limpieza de recursos

```bash
# Eliminar API Gateway
aws apigateway delete-rest-api --rest-api-id <API_ID>

# Eliminar Lambdas
aws lambda delete-function --function-name escolaronline-procesar-carrito
aws lambda delete-function --function-name escolaronline-crear-orden
aws lambda delete-function --function-name escolaronline-procesar-orden

# Eliminar DynamoDB
aws dynamodb delete-table --table-name carrito-escolaronline

# Eliminar SQS
aws sqs delete-queue --queue-url <QUEUE_URL>
aws sqs delete-queue --queue-url <DLQ_URL>

# Eliminar S3 (vaciar primero)
aws s3 rm s3://escolaronline-frontend-<ACCOUNT_ID> --recursive
aws s3 rb s3://escolaronline-frontend-<ACCOUNT_ID>

# Eliminar VPC Endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <ENDPOINT_ID>
```

---

2026 - Disenador: Ignacio A. Pastenet M.
