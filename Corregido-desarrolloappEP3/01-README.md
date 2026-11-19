# FreshBox SpA — Arquitectura Hibrida Serverless (EP3)

## Descripcion

Evolucion de FreshBox hacia modelo hibrido: EKS (4 pods CRUD productos) + Serverless (carrito + ordenes). Agrega carrito de compras y procesamiento desacoplado de ordenes.

## Referencias Paso a Paso De Prueba EKS
<p>Crear VPC:  6 subredes, 2 publicas, 4 privadas <br>
    Subredes publicas activar poder generar IPsv4 publicas para el eks<br>
    Subred privada 1 y 2 Capa App <br>
    Subred privada 3 y 4 Capa Datos<br>
    Crear Cluster de EKS sin modo automatico <br>
    Crear Grupos de Nodos para el EKS con t4g.xxxx <br>
    <br>
    Crear Grupo de subredes para la RDS 
    <br>
    Crear Security Group de RDS (puede dejar en blanco al momento y luego modificar)<br>
    <br>
    Crear Mysql RDS usando grupo subred y SG antes generado<br>
    <br>
    Una vez terminado el cluster con los nodos implementar la configuracion del Ingress antes de desplegar (realice inmediatamente incluso la parte del error, se adjunto manual en los archivos)<br>
    <br>
    Si necesita crear una EC2, realicelo en la subred privada 1 o 2 y evalue los permisos necesarios en los security group para comunicacion.<br>
    <br>
    Asegurece de que el security group del RDS tiene como regla de entrada (inboud rule) el acceso al puerto 3306 desde el security group del cluster (referencia de nombre de este security group eks-cluster-sg-cluster0XXXXXXXXXX), sino al desplegar las aplicaciones de microservicios no podran comunicarse con la base de datos <br>
    <br>
    Necesita cambiar la url del endpoint del RDS en el archivo deployments.yaml <br>
    <br>
    TIENE QUE CREAR LA BASE DE DATOS SON SU INFORMACION, SE ADJUNTO init.sql CON LO NECESARIO PARA ESTO.<br>
    <br>
    Configure su cuenta de AWS<br>
    <br>
    Conectese al cluster (cambie el nombre del cluster)<br>
    aws eks update-kubeconfig --region us-east-1 --name cluster01 <br>
    <br>
    Verifique conexion al cluster y Despliegue el archivo deployments.yaml<br>
    kubectl get nodes <br>
    kubectl get pods -A <br>
    kubectl apply -f deployments.yaml<br>
    <br>
    Verificar que el ingress da una url publica<br>
    kubectl get ingress -n fresbox
    
 </p>

### DESPUES DE IMPLEMENTAR LOS SERVICIOS SERVERLESS TENDRA QUE VALIDAR NUEVAMENTE LOS SECURITY GROUPS YA QUE EL CLUSTER EKS GENERA 3 SECURITY GROUPS

## Arquitectura EP3 (Hibrida: EKS + Serverless)

| Capa | Componente | Servicio AWS |
|------|-----------|--------------|
| Frontend | Sitio estatico | S3 Static Website |
| API | Fachada unica | API Gateway REST |
| Contenedores | CRUD Productos (4 pods) | EKS → RDS MySQL |
| Serverless | Carrito | Lambda Cart → DynamoDB |
| Serverless | Crear orden | Lambda Create-Order → SQS |
| Serverless | Procesar orden | Lambda Process-Order → MySQL |
| Mensajeria | Cola ordenes | SQS + DLQ |
| BD NoSQL | Carrito temporal | DynamoDB (TTL 24h) |
| BD Relacional | Productos + ordenes | RDS MySQL Multi-AZ |

## Estructura

```
desarrolloappEP3/
├── README.md
├── init.sql                        (schema: productos + ordenes)
├── microserviciosEKSBackend/
│   ├── get-products/  (Dockerfile, index.js, package.json)
│   ├── create-product/
│   ├── update-product/
│   └── delete-product/
├── S3Frontend/                     (HTML/CSS/JS para S3 con carrito + ordenes)
│   ├── index.html
│   ├── css/styles.css
│   └── js/app.js
├── lambda/
│   ├── procesar-carrito/           (DynamoDB)
│   │   ├── index.js
│   │   └── package.json
│   ├── crear-orden/                (DynamoDB + SQS)
│   │   ├── index.js
│   │   └── package.json
│   └── procesar-orden/             (SQS → MySQL)
│       ├── index.js
│       └── package.json
├── k8s/
│   └── deployments.yaml            (4 pods + services + ingress + HPA)
└── scripts/
    ├── ecr-push.sh
    └── guia-docente-ep3.md
```

## Variables de Entorno

| Variable | Valor |
|----------|-------|
| DB_HOST | (RDS endpoint) |
| DB_USER | alumno |
| DB_PASS | alumno123 |
| DB_NAME | freshbox |
| TABLE_NAME | carrito-freshbox |
| QUEUE_URL | (SQS URL) |

## Nota

- El desarrollo es **autocontenido**: incluye microservicios EKS, frontend S3, Lambdas, manifiestos K8s y scripts
- Frontend va en S3 (no en pod EKS)
- Se agregan 3 Lambdas (carrito + ordenes)
- DynamoDB, SQS, API Gateway se configuran en consola AWS

---

2026 - Disenador: Ignacio A. Pastenet M.
