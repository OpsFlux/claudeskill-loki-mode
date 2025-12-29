# 部署参考

所有支持平台的基础设施配置和部署说明。

## 部署决策矩阵

| 标准 | Vercel/Netlify | Railway/Render | AWS | GCP | Azure |
|----------|----------------|----------------|-----|-----|-------|
| 静态/JAMstack | 最佳 | 良好 | 大材小用 | 大材小用 | 大材小用 |
| 简单全栈 | 良好 | 最佳 | 大材小用 | 大材小用 | 大材小用 |
| 扩展到数百万用户 | 不支持 | 有限 | 最佳 | 最佳 | 最佳 |
| 企业合规 | 有限 | 有限 | 最佳 | 良好 | 最佳 |
| 大规模成本 | 昂贵 | 适中 | 最便宜 | 便宜 | 适中 |
| 设置复杂度 | 微不足道 | 容易 | 复杂 | 复杂 | 复杂 |

## 快速开始命令

### Vercel
```bash
# 安装 CLI
npm i -g vercel

# 部署（自动检测框架）
vercel --prod

# 环境变量
vercel env add VARIABLE_NAME production
```

### Netlify
```bash
# 安装 CLI
npm i -g netlify-cli

# 部署
netlify deploy --prod

# 环境变量
netlify env:set VARIABLE_NAME value
```

### Railway
```bash
# 安装 CLI
npm i -g @railway/cli

# 登录并部署
railway login
railway init
railway up

# 环境变量
railway variables set VARIABLE_NAME=value
```

### Render
```yaml
# render.yaml（基础设施即代码）
services:
  - type: web
    name: api
    env: node
    buildCommand: npm install && npm run build
    startCommand: npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        fromDatabase:
          name: postgres
          property: connectionString

databases:
  - name: postgres
    plan: starter
```

---

## AWS 部署

### 架构模板
```
┌─────────────────────────────────────────────────────────┐
│                        CloudFront                        │
└─────────────────────────┬───────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          │                               │
    ┌─────▼─────┐                   ┌─────▼─────┐
    │    S3     │                   │    ALB    │
    │ (static)  │                   │           │
    └───────────┘                   └─────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │   ECS     │
                                    │  Fargate  │
                                    └─────┬─────┘
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                        ┌─────▼─────┐           ┌─────▼─────┐
                        │    RDS    │           │ ElastiCache│
                        │ Postgres  │           │   Redis   │
                        └───────────┘           └───────────┘
```

### Terraform 配置
```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "terraform-state-${var.project_name}"
    key    = "state.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.aws_region}a", "${var.aws_region}b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
}

# ECS 集群
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# RDS
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "6.0.0"

  identifier = "${var.project_name}-db"

  engine               = "postgres"
  engine_version       = "15"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = var.environment == "production" ? "db.t3.medium" : "db.t3.micro"

  allocated_storage = 20
  storage_encrypted = true

  db_name  = var.db_name
  username = var.db_username
  port     = 5432

  vpc_security_group_ids = [aws_security_group.rds.id]
  subnet_ids             = module.vpc.private_subnets

  backup_retention_period = var.environment == "production" ? 7 : 1
  deletion_protection     = var.environment == "production"
}
```

### ECS 任务定义
```json
{
  "family": "app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "${ECR_REPO}:${TAG}",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "NODE_ENV", "value": "production"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

### GitHub Actions CI/CD
```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: app
  ECS_SERVICE: app-service
  ECS_CLUSTER: app-cluster

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-definition.json
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

---

## GCP 部署

### Cloud Run（大多数情况下推荐）
```bash
# 构建和部署
gcloud builds submit --tag gcr.io/PROJECT_ID/app
gcloud run deploy app \
  --image gcr.io/PROJECT_ID/app \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars="NODE_ENV=production" \
  --set-secrets="DATABASE_URL=db-url:latest"
```

### GCP 的 Terraform
```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}

# Cloud Run 服务
resource "google_cloud_run_service" "app" {
  name     = "app"
  location = var.region

  template {
    spec {
      containers {
        image = "gcr.io/${var.project_id}/app:latest"

        ports {
          container_port = 3000
        }

        env {
          name  = "NODE_ENV"
          value = "production"
        }

        env {
          name = "DATABASE_URL"
          value_from {
            secret_key_ref {
              name = google_secret_manager_secret.db_url.secret_id
              key  = "latest"
            }
          }
        }

        resources {
          limits = {
            cpu    = "1000m"
            memory = "512Mi"
          }
        }
      }
    }

    metadata {
      annotations = {
        "autoscaling.knative.dev/maxScale" = "10"
        "run.googleapis.com/cloudsql-instances" = google_sql_database_instance.main.connection_name
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

# Cloud SQL
resource "google_sql_database_instance" "main" {
  name             = "app-db"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier = "db-f1-micro"

    backup_configuration {
      enabled = true
    }
  }

  deletion_protection = var.environment == "production"
}
```

---

## Azure 部署

### Azure Container Apps
```bash
# 创建资源组
az group create --name app-rg --location eastus

# 创建 Container Apps 环境
az containerapp env create \
  --name app-env \
  --resource-group app-rg \
  --location eastus

# 部署容器
az containerapp create \
  --name app \
  --resource-group app-rg \
  --environment app-env \
  --image myregistry.azurecr.io/app:latest \
  --target-port 3000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --env-vars "NODE_ENV=production"
```

---

## Kubernetes 部署

### 清单
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: app:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

### Helm Chart 结构
```
chart/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── hpa.yaml
```

---

## 蓝绿部署

### 策略
```
1. 将新版本部署到"绿色"环境
2. 对绿色环境运行冒烟测试
3. 将负载均衡器切换到绿色环境
4. 监控 15 分钟
5. 如果健康：停用蓝色环境
6. 如果出错：切换回蓝色环境（回滚）
```

### 实现 (AWS ALB)
```bash
# 部署绿色环境
aws ecs update-service --cluster app --service app-green --task-definition app:NEW_VERSION

# 等待稳定
aws ecs wait services-stable --cluster app --services app-green

# 运行冒烟测试
curl -f https://green.app.example.com/health

# 切换流量（更新目标组权重）
aws elbv2 modify-listener-rule \
  --rule-arn $RULE_ARN \
  --actions '[{"Type":"forward","TargetGroupArn":"'$GREEN_TG'","Weight":100}]'
```

---

## 回滚程序

### 立即回滚
```bash
# AWS ECS
aws ecs update-service --cluster app --service app --task-definition app:PREVIOUS_VERSION

# Kubernetes
kubectl rollout undo deployment/app

# Vercel
vercel rollback
```

### 自动回滚触发器
部署后监控这些指标：
- 错误率 > 1% 持续 5 分钟
- p99 延迟 > 500ms 持续 5 分钟
- 健康检查失败 > 3 次连续
- 内存使用 > 90% 持续 10 分钟

如果触发任何一个指标，执行自动回滚。

---

## 密钥管理

### AWS Secrets Manager
```bash
# 创建密钥
aws secretsmanager create-secret \
  --name app/database-url \
  --secret-string "postgresql://..."

# 在 ECS 任务中引用
"secrets": [
  {
    "name": "DATABASE_URL",
    "valueFrom": "arn:aws:secretsmanager:region:account:secret:app/database-url"
  }
]
```

### HashiCorp Vault
```bash
# 存储密钥
vault kv put secret/app database-url="postgresql://..."

# 在应用中读取
vault kv get -field=database-url secret/app
```

### 环境特定
```
.env.development   # 本地开发
.env.staging       # 预发布环境
.env.production    # 生产环境（永远不要提交）
```

所有生产密钥必须在密钥管理器中，绝不能在代码或环境文件中。
