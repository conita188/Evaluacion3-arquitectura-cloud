# Ingress EKS

### Primero taguear las subredes publicas y privadas con 

                           
<p>TAG/Etiqueta Tipo de subnet	Pública<br>
kubernetes.io/role/elb = 1 <br>
</p>

<p>TAG/Etiqueta en subnet	Privada<br>
kubernetes.io/role/internal-elb = 1<br>
</p>
   
### Realizar la instalacion del AWS Load Balancer Complemento
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.12.0/cert-manager.yaml

  
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   
chmod 700 get_helm.sh 

./get_helm.sh 

helm repo add eks https://aws.github.io/eks-charts

helm repo update

kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"


#### Aqui cambiar cluster01 por el nombre de su cluster de EKS
aws eks describe-cluster --name cluster01 --region us-east-1 --query "cluster.resourcesVpcConfig.vpcId" --output text

#### EL comando anterior da la id de la VPC para aplicar el cambio del siguiente comando
  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName="cluster01"   --set region="us-east-1"   --set vpcId="vpc-1234567890"
 
kubectl get deployment -n kube-system aws-load-balancer-controller
 
kubectl get ingressclass

### Despues de que despliegue deployement.yml comprobar que sea crea la url publica

kubectl get ingress -n escolaronline

### En caso de que no se muestre url (puede salir *) comprobar si sale el siguiente error

```
kubectl describe ingress escolaronline-ingress -n escolaronline

Name:             escolaronline-ingress
Labels:           <none>
Namespace:        escolaronline
Address:          
Ingress Class:    alb
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /api/products   get-products:3001 (10.0.134.183:3001,10.0.154.88:3001)
Annotations:  alb.ingress.kubernetes.io/healthcheck-path: /api/products
              alb.ingress.kubernetes.io/scheme: internet-facing
              alb.ingress.kubernetes.io/target-type: ip
Events:
  Type     Reason            Age                   From     Message
  ----     ------            ----                  ----     -------
  Warning  FailedBuildModel  3m14s (x22 over 70m)  ingress  Failed build model due to operation error Elastic Load Balancing v2: DescribeLoadBalancers, get identity: get credentials: failed to refresh cached credentials, no EC2 IMDS role found, operation error ec2imds: GetMetadata, canceled, context deadline exceeded
```

### Realizar el siguiente arreglo desde una shell con acceso a la API de AWS

```
export CLUSTER_NAME=cluster01
export AWS_REGION=us-east-1

for ID in $(aws ec2 describe-instances --region $AWS_REGION \
  --filters "Name=tag:eks:cluster-name,Values=$CLUSTER_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text); do   echo "Modificando $ID";   aws ec2 modify-instance-metadata-options --region $AWS_REGION     --instance-id $ID --http-put-response-hop-limit 2 --http-tokens required; done

aws ec2 describe-instances --region $AWS_REGION   --filters "Name=tag:eks:cluster-name,Values=$CLUSTER_NAME"   --query "Reservations[].Instances[].MetadataOptions.HttpPutResponseHopLimit"

kubectl rollout restart deployment aws-load-balancer-controller -n kube-system

kubectl rollout status deployment aws-load-balancer-controller -n kube-system

kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=30

kubectl get pods -n kube-system | grep load-balancer

kubectl get ingress -n escolaronline -w
```

# EL INGRESS GENERA SECURITY GROUPS APARTE DE LOS CONFIGURADOS EN EL EKS, UNA COSA ES LA CONEXION DESDE UNA MAQUINA EN LA MISMA RED Y OTRA DESDE LOS PODS, REVISAR ESA CONFIGURACION DESPUES DE HACER EL DEPLOYMENT DE TODAS LAS CONFIGURACIONES DE 