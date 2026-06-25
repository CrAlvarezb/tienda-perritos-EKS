# Informe de Evaluación — Tienda Perritos EKS

## 1. Demostración del Pipeline CI/CD con Cambios de Color

### 1.1 Flujo Completo

Se realizaron **4 cambios consecutivos** al color del header del frontend, cada uno demostrando el ciclo completo de CI/CD:

| # | Color | Código | Commit | Pipeline |
|---|---|---|---|---|
| 1 | Azul → Verde | `#3b82f6` → `#10b981` | `fix: cambiar color header a verde` | ✅ Success |
| 2 | Verde → Dorado | `#10b981` → `#f59e0b` | `fix: cambiar header a dorado` | ✅ Success |
| 3 | Dorado → Rosado | `#f59e0b` → `#ec4899` | `fix: cambiar header a rosado` | ✅ Success |
| 4 | Rosado → Negro | `#ec4899` → `#1f2937` | `fix: cambiar header a negro` | ⏳ Running |

### 1.2 Ciclo del Pipeline (por cada cambio)

```
1. EDITAR código (frontend/index.html)
        ↓
2. git add + git commit + git push origin main
        ↓
3. GitHub Actions detecta el push a main
        ↓
4. Build de la imagen Docker con el nuevo color
        ↓
5. Push de la imagen a Amazon ECR
        ↓
6. Reemplazo de {{ECR_URL}} en los manifests
        ↓
7. kubectl apply → despliegue en EKS
        ↓
8. Nuevo pod con el color actualizado
        ↓
9. URL pública refleja el cambio
```

### 1.3 Evidencia de funcionamiento

- **URL pública:** http://a3d8a2621fdeb474685bd9e62918ee17-440057208.us-east-1.elb.amazonaws.com
- **Pipeline:** https://github.com/CrAlvarezb/tienda-perritos-EKS/actions
- **API productos:** `GET /api/productos` → 200 OK con 5 productos

---

## 2. Infraestructura AWS

### 2.1 Cluster EKS
- Cluster `tienda-perritos` (K8s 1.31)
- Nodegroup `tienda-nodes`: 2 nodos `t3.medium` (escalable 1-3)
- Región: `us-east-1`

### 2.2 VPC y Redes
- VPC: `vpc-0ab3b425dce2f5f74`
- 5 subnets públicas en `us-east-1a, b, c, d, f`
- Internet Gateway conectado
- Security Groups: administrados por EKS

### 2.3 Roles IAM
- `LabEksClusterRole`: permisos del plano de control
- `LabEksNodeRole`: permisos de nodos worker (ECR, CloudWatch, EC2)

### 2.4 Amazon ECR
| Repositorio | URI |
|---|---|
| tienda-frontend | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend` |
| tienda-backend | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-backend` |
| tienda-db | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-db` |

---

## 3. Servicios Desplegados

### 3.1 Frontend
- 2 réplicas, type LoadBalancer (público)
- Nginx con proxy reverso a backend por DNS interno
- Health probes: readiness y liveness

### 3.2 Backend
- 2 réplicas, type ClusterIP (interno)
- Node.js + Express en puerto 3001
- Variables de entorno desde Secrets de K8s
- Comunicación con MySQL vía DNS interno `tienda-db:3306`

### 3.3 Base de Datos
- 1 réplica MySQL 8
- Headless service (ClusterIP: None)
- Secret con password en base64

---

## 4. Autoscaling (HPA)

| Componente | Métrica | Target | Réplicas |
|---|---|---|---|
| Backend | CPU | 70% | 2 - 10 |
| Frontend | CPU | 60% | 2 - 6 |

**Justificación:**
- **Backend 70%:** Node.js procesa consultas a MySQL. Al escalar al 70% se evitan escalados prematuros, optimizando recursos.
- **Frontend 60%:** Nginx maneja conexiones concurrentes. Escalar al 60% asegura respuesta rápida al usuario.

---

## 5. Comunicación Front → Back

El frontend no expone el backend directamente. Usa Nginx como proxy reverso:

```nginx
location /api/ {
    proxy_pass http://tienda-backend:3001;
}
```

El frontend llama a `/api/productos` y Nginx resuelve el DNS interno `tienda-backend:3001` (ClusterIP).

---

## 6. Pipeline CI/CD — Detalle

### Archivo: `.github/workflows/deploy-eks.yml`

**Disparadores:**
- Push a rama `main`
- Ejecución manual (`workflow_dispatch`)

**Secretos requeridos (GitHub → Settings → Secrets → Actions):**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`

**Pasos:**
1. Checkout del repositorio
2. Configuración de credenciales AWS
3. Login a Amazon ECR
4. Build y push de imágenes (frontend, backend, db) con tags SHA + latest
5. Instalación de kubectl
6. Configuración de kubeconfig contra EKS
7. Reemplazo de `{{ECR_URL}}` por la URL real del registry
8. Despliegue de MySQL (secret + deployment + service)
9. Despliegue de Backend (deployment + service)
10. Despliegue de Frontend (deployment + service)
11. Aplicación de HPA
12. Obtención de URL del LoadBalancer

---

## 7. Pruebas Realizadas

### API
```bash
curl http://a3d8a2621fdeb474685bd9e62918ee17-440057208.us-east-1.elb.amazonaws.com/api/productos
# → 200 OK
# → [5 productos con id, nombre, precio, stock]
```

### Logs
```bash
kubectl logs -n tienda -l app=tienda-backend --tail=50
```

### Recuperación
```bash
kubectl rollout restart deployment tienda-frontend -n tienda
# Pods se reemplazan sin downtime
```

---

## 8. Problemas Encontrados y Soluciones

| Problema | Solución |
|---|---|
| Pipeline falló por falta de AWS Secrets | Configurar secrets en GitHub |
| Pods no actualizaban imagen `:latest` tras pipeline | `kubectl rollout restart deployment` |
| Actions mostraba commits de Basty66 | Crear orphan branch con historial limpio |
| Cambios en rama equivocada (evaluation vs main) | Mergear evaluation → main para triggerear pipeline |

---

## 9. Diagrama de Arquitectura Final

```
                    ┌──────────────┐
                    │   Usuario    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  AWS ALB     │
                    │  (puerto 80) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Frontend   │  x2 pods
                    │   Nginx      │  t3.medium
                    └──────┬───────┘
                           │ proxy_pass
                    ┌──────▼───────┐
                    │   Backend    │  x2 pods
                    │   Node.js    │  puerto 3001
                    │   Express    │  ClusterIP
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │    MySQL     │  x1 pod
                    │    puerto    │  Headless
                    │    3306      │  Service
                    └──────────────┘
```

---

*Documento generado para evaluación EP3 — Tienda Perritos EKS*
