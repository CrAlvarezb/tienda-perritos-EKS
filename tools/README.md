# Tools - Crear EKS Cluster

## Prerequisitos
```bash
aws configure set region us-east-1
```

## 1. Verificar roles y subnets (cambian por sesion)
```bash
# Subnets disponibles (excluir us-east-1e)
aws ec2 describe-subnets --query 'Subnets[?AvailabilityZone!=`us-east-1e`].[SubnetId,AvailabilityZone]' --output text

# Role del cluster
aws iam list-roles --query "Roles[?contains(RoleName,'LabEksClusterRole')].Arn" --output text

# Role de nodos
aws iam list-roles --query "Roles[?contains(RoleName,'LabEksNodeRole')].Arn" --output text
```

## 2. Actualizar cluster.json y nodegroup.json
Editar los ARN de roles y subnetIds segun el output de arriba.

## 3. Crear cluster (~10 min)
```bash
aws eks create-cluster --cli-input-json "file://cluster.json" --region us-east-1
aws eks wait cluster-active --name tienda-perritos --region us-east-1
```

## 4. Crear nodegroup (~3 min)
```bash
aws eks create-nodegroup --cli-input-json "file://nodegroup.json" --region us-east-1
aws eks wait nodegroup-active --cluster-name tienda-perritos --nodegroup-name tienda-nodes --region us-east-1
```

## 5. Configurar kubectl
```bash
aws eks update-kubeconfig --name tienda-perritos --region us-east-1
kubectl get nodes
```

## 6. Crear ECR repos
```bash
aws ecr create-repository --repository-name tienda-frontend --region us-east-1
aws ecr create-repository --repository-name tienda-backend --region us-east-1
aws ecr create-repository --repository-name tienda-db --region us-east-1
```

## 7. Tirar pipeline
```bash
gh workflow run deploy-eks.yml --repo CrAlvarezb/tienda-perritos-EKS
```
