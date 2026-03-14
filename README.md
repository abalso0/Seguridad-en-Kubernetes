# Demo 2: IRSA Cada pod con sus propios permisos

### 1. Habilitar OIDC en el cluster

```bash
eksctl utils associate-iam-oidc-provider --cluster test --approve
```

### 2. Obtener OIDC issuer y desplegar CloudFormation

```bash
OIDC=$(aws eks describe-cluster --name test \
  --query cluster.identity.oidc.issuer --output text | sed 's|https://||')

aws cloudformation deploy \
  --template-file irsa-stack.yaml \
  --stack-name eks-irsa-demo \
  --parameter-overrides ClusterOIDCIssuer=$OIDC \
  --capabilities CAPABILITY_NAMED_IAM
```

### 3. Crear namespace y service account

```bash
ROLE_ARN=$(aws cloudformation describe-stacks --stack-name eks-irsa-demo \
  --query "Stacks[0].Outputs[?OutputKey=='RoleArn'].OutputValue" --output text)

kubectl create namespace irsa-demo
kubectl create serviceaccount s3-reader-sa -n irsa-demo
kubectl annotate sa s3-reader-sa -n irsa-demo eks.amazonaws.com/role-arn=$ROLE_ARN
```

### 4. Subir archivo de prueba al bucket

```bash
BUCKET=$(aws cloudformation describe-stacks --stack-name eks-irsa-demo \
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" --output text)

echo "datos-sensibles" | aws s3 cp - s3://$BUCKET/test.txt
```

---

## Demo 

### Paso 1: Pod SIN IRSA, mostrar el problema

```bash
kubectl run pod-sin-irsa -n irsa-demo --image=amazon/aws-cli --command -- sleep 3600
kubectl wait --for=condition=Ready pod/pod-sin-irsa -n irsa-demo --timeout=60s

kubectl exec -n irsa-demo pod-sin-irsa -- env | grep AWS
kubectl exec -n irsa-demo pod-sin-irsa -- aws sts get-caller-identity 2>&1
```

**DECIR:** "Este pod no tiene identidad propia. Usa las credenciales del nodo, o no tiene ninguna. Todos los pods comparten lo mismo."

### Paso 2: Pod CON IRSA - mostrar la solución

```bash
kubectl run pod-con-irsa -n irsa-demo --image=amazon/aws-cli \
  --overrides='{"spec":{"serviceAccountName":"s3-reader-sa","containers":[{"name":"pod-con-irsa","image":"amazon/aws-cli","command":["sleep","3600"]}]}}'

kubectl wait --for=condition=Ready pod/pod-con-irsa -n irsa-demo --timeout=60s

kubectl exec -n irsa-demo pod-con-irsa -- env | grep AWS
```

**Credenciales:**
```
AWS_ROLE_ARN=arn:aws:iam::<ACCOUNT>:role/eks-s3-reader-role
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/...
```

### "EKS inyectó credenciales temporales automáticamente. Este pod tiene su propia identidad."

### Paso 3: Probar permisos - least privilege

```bash
# Puede LEER 
kubectl exec -n irsa-demo pod-con-irsa -- aws s3 ls s3://$BUCKET

# NO puede ESCRIBIR 
kubectl exec -n irsa-demo pod-con-irsa -- aws s3 cp /etc/hostname s3://$BUCKET/hack.txt 2>&1

# Tiene identidad propia
kubectl exec -n irsa-demo pod-con-irsa -- aws sts get-caller-identity
```



---

## Eliminar recursos

```bash
kubectl delete namespace irsa-demo
aws cloudformation delete-stack --stack-name eks-irsa-demo
```
