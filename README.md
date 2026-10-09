# HUNCHOGitHub repo
├── backend/                    ← FastAPI app
│   ├── railway.json            ← deployment config
│   └── ...
├── frontend/                   ← Vite React app
│   ├── railway.json
│   └── ...
├── infrastructure/
│   ├── main.tf                 ← Terraform: Railway project, services, volumes
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars.example
└── .github/workflows/
    ├── ci.yml                  ← lint + test on PR
    ├── deploy.yml              ← build, migrate, deploy, verify
    └── railway-config.yml      ← IaC drift detection

{
  "$schema": "https://railway.com/railway.schema.json",
  "build": {
    "builder": "RAILPACK",
    "buildCommand": "pip install -r requirements.txt"
  },
  "deploy": {
    "startCommand": "uvicorn main:app --host 0.0.0.0 --port $PORT",
    "preDeployCommand": ["python -m scripts.migrate"],
    "healthcheckPath": "/health/readiness",
    "healthcheckTimeout": 300,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}

https://railway.com/railway.schema.json

import time
import psutil
from fastapi import APIRouter

router = APIRouter(prefix="/health", tags=["health"])
_start_time = time.time()


@router.get("/liveness")
def liveness():
    """Process is alive. Never check external deps here."""
    return {"status": "up", "uptime_seconds": time.time() - _start_time}


@router.get("/readiness")
def readiness():
    """Ready to serve traffic. Check critical dependencies here."""
    checks = {}
    try:
        from database import SessionLocal
        from sqlalchemy import text
        db = SessionLocal()
        try:
            db.execute(text("SELECT 1"))
            checks["database"] = "healthy"
        finally:
            db.close()
    except Exception as e:
        checks["database"] = f"unhealthy: {e}"
        return JSONResponse(status_code=503, content={"status": "not ready", "checks": checks})

    # Optional: check memory pressure
    mem = psutil.virtual_memory()
    if mem.percent > 95:
        checks["memory"] = f"critical: {mem.percent}%"
        return JSONResponse(status_code=503, content={"status": "not ready", "checks": checks})
    checks["memory"] = "ok"

    return {"status": "ready", "checks": checks}


app.include_router(router)

terraform {
  required_providers {
    railway = {
      source  = "terraform-community-providers/railway"
      version = "~> 0.4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6.0"
    }
  }
}

provider "railway" {
  token = var.railway_token
}

# ---------- Project ----------

resource "railway_project" "vault" {
  name = "secure-vault"
}

# ---------- Shared secrets ----------

resource "random_password" "jwt_secret" {
  length  = 64
  special = false
}

resource "random_password" "fernet_key" {
  length  = 44
  special = true
}

# ---------- PostgreSQL ----------

resource "railway_service" "postgres" {
  project_id = railway_project.vault.id
  name       = "postgres"
  source_image = "ghcr.io/railwayapp-templates/postgres-ssl:16"
}

resource "railway_volume" "postgres_data" {
  project_id = railway_project.vault.id
  service_id = railway_service.postgres.id
  mount_path = "/var/lib/postgresql/data"
}

# ---------- Backend service ----------

resource "railway_service" "backend" {
  project_id = railway_project.vault.id
  name       = "backend"
  source_repo = var.backend_repo
  source_branch = "main"
  root_directory = "backend"
}

resource "railway_variable" "backend_vars" {
  for_each = {
    DATABASE_URL               = "postgresql://postgres:${railway_variable.postgres_password.value}@${railway_service.postgres.internal_domain}:5432/railway"
    JWT_SECRET                 = random_password.jwt_secret.result
    JWT_ALGORITHM              = "HS256"
    JWT_EXPIRE_MINUTES         = "60"
    FERNET_KEY                 = random_password.fernet_key.result
    STRIPE_SECRET_KEY          = var.stripe_secret_key
    STRIPE_WEBHOOK_SECRET      = var.stripe_webhook_secret
    STRIPE_CONNECT_CLIENT_ID   = var.stripe_connect_client_id
    STRIPE_SUCCESS_URL         = "https://${railway_service.frontend.public_domain}/success"
    STRIPE_CANCEL_URL          = "https://${railway_service.frontend.public_domain}/cancel"
    STRIPE_PLATFORM_FEE_BPS    = "250"
    STRIPE_PLATFORM_FEE_MIN_CENTS = "50"
    STRIPE_PLATFORM_FEE_MAX_CENTS = "5000"
    ADMIN_EMAIL                = var.admin_email
    ADMIN_PASSWORD             = var.admin_password
    SMTP_HOST                  = var.smtp_host
    SMTP_PORT                  = var.smtp_port
    SMTP_USER                  = var.smtp_user
    SMTP_PASSWORD              = var.smtp_password
    SMTP_FROM                  = var.smtp_from
    APP_BASE_URL               = "https://${railway_service.frontend.public_domain}"
    SENTRY_DSN                 = var.sentry_dsn
    SLACK_WEBHOOK_URL          = var.slack_webhook_url
    WITHDRAWAL_AUTO_APPROVE_LIMIT = "500"
    WITHDRAWAL_REQUIRED_APPROVALS = "2"
    EMAIL_VERIFY_EXPIRE_HOURS  = "24"
    ENV                        = "production"
  }
  project_id = railway_project.vault.id
  service_id = railway_service.backend.id
  key        = each.key
  value      = each.value
}

# ---------- Frontend service ----------

resource "railway_service" "frontend" {
  project_id    = railway_project.vault.id
  name          = "frontend"
  source_repo   = var.frontend_repo
  source_branch = "main"
  root_directory = "frontend"
}

resource "railway_variable" "frontend_vars" {
  project_id = railway_project.vault.id
  service_id = railway_service.frontend.id
  key        = "VITE_API_BASE"
  value      = "https://${railway_service.backend.public_domain}"
}

# ---------- Postgres password ----------

resource "random_password" "postgres_password" {
  length  = 32
  special = false
}

resource "railway_variable" "postgres_password" {
  project_id = railway_project.vault.id
  service_id = railway_service.postgres.id
  key        = "POSTGRES_PASSWORD"
  value      = random_password.postgres_password.result
}

variable "railway_token" {
  description = "Railway API token (account or workspace scoped)"
  type        = string
  sensitive   = true
}

variable "backend_repo" {
  description = "GitHub repo for backend (e.g. youruser/vault-backend)"
  type        = string
}

variable "frontend_repo" {
  description = "GitHub repo for frontend"
  type        = string
}

variable "stripe_secret_key" {
  type      = string
  sensitive = true
}

variable "stripe_webhook_secret" {
  type      = string
  sensitive = true
}

variable "stripe_connect_client_id" {
  type = string
}

variable "admin_email" {
  type = string
}

variable "admin_password" {
  type      = string
  sensitive = true
}

variable "smtp_host" {
  type    = string
  default = ""
}

variable "smtp_port" {
  type    = string
  default = "587"
}

variable "smtp_user" {
  type    = string
  default = ""
}

variable "smtp_password" {
  type      = string
  sensitive = true
  default   = ""
}

variable "smtp_from" {
  type    = string
  default = "noreply@yourdomain.com"
}

variable "sentry_dsn" {
  type    = string
  default = ""
}

variable "slack_webhook_url" {
  type    = string
  default = ""
}

output "backend_url" {
  value = "https://${railway_service.backend.public_domain}"
}

output "frontend_url" {
  value = "https://${railway_service.frontend.public_domain}"
}

output "postgres_internal_url" {
  value     = "postgresql://postgres:${random_password.postgres_password.result}@${railway_service.postgres.internal_domain}:5432/railway"
  sensitive = true
}

output "jwt_secret" {
  value     = random_password.jwt_secret.result
  sensitive = true
}

railway_token                = "your-railway-api-token"
backend_repo                 = "youruser/vault-backend"
frontend_repo                = "youruser/vault-frontend"
stripe_secret_key            = "sk_live_..."
stripe_webhook_secret        = "whsec_..."
stripe_connect_client_id     = "ca_..."
admin_email                  = "admin@yourdomain.com"
admin_password               = "StrongPassword!123"
smtp_host                    = "smtp.postmarkapp.com"
smtp_user                    = "your-postmark-token"
smtp_password                = "your-postmark-token"
smtp_from                    = "noreply@yourdomain.com"
sentry_dsn                   = "https://...@sentry.io/..."
slack_webhook_url            = "https://hooks.slack.com/services/..."

name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"
          cache-dependency-path: backend/requirements.txt

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install black ruff pytest pytest-asyncio httpx

      - name: Lint
        run: |
          black --check .
          ruff check .

      - name: Test
        env:
          JWT_SECRET: "test-secret-that-is-long-enough-for-validation"
          FERNET_KEY: "dGVzdC1rZXktZm9yLXRlc3Rpbmctb25seS1ub3QtcHJvZA=="
          DATABASE_URL: "sqlite:///./test.db"
          STRIPE_SECRET_KEY: "sk_test_dummy"
          STRIPE_WEBHOOK_SECRET: "whsec_dummy"
          ADMIN_EMAIL: "admin@test.local"
          ADMIN_PASSWORD: "TestPassword123!"
        run: pytest -v --tb=short

      - name: Save OpenAPI snapshot
        if: always()
        env:
          JWT_SECRET: "test-secret-that-is-long-enough-for-validation"
          FERNET_KEY: "dGVzdC1rZXktZm9yLXRlc3Rpbmctb25seS1ub3QtcHJvZA=="
          DATABASE_URL: "sqlite:///./test.db"
        run: |
          python -c "
          from main import app
          import json
          schema = app.openapi()
          with open('openapi-snapshot.json', 'w') as f:
              json.dump(schema, f, indent=2)
          print(f'OpenAPI schema saved: {len(schema[\"paths\"])} endpoints')
          "

      - name: Upload OpenAPI artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: openapi-snapshot
          path: backend/openapi-snapshot.json
          retention-days: 30

  frontend-tests:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run build



name: Deploy to Railway

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Railway CLI
        run: npm install -g @railway/cli

      - name: Deploy backend
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: |
          cd backend
          railway up --ci --service backend
        # --ci streams build logs and exits when build completes [citation:2]

      - name: Verify backend health
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
          BACKEND_URL: ${{ secrets.BACKEND_URL }}
        run: |
          for i in {1..30}; do
            if curl -sf "$BACKEND_URL/health/readiness" | grep -q '"status":"ready"'; then
              echo "Backend is ready"
              exit 0
            fi
            echo "Attempt $i: not ready yet"
            sleep 10
          done
          echo "Backend failed to become healthy"
          exit 1

      - name: Rollback on failure
        if: failure()
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: |
          # Re-deploy the previous successful deployment
          railway redeploy --service backend --yes

  deploy-frontend:
    runs-on: ubuntu-latest
    needs: deploy-backend
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Install Railway CLI
        run: npm install -g @railway/cli

      - name: Deploy frontend
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: |
          cd frontend
          railway up --ci --service frontend

      - name: Verify frontend
        env:
          FRONTEND_URL: ${{ secrets.FRONTEND_URL }}
        run: |
          for i in {1..20}; do
            if curl -sf "$FRONTEND_URL" | grep -q "Secure Vault"; then
              echo "Frontend is live"
              exit 0
            fi
            sleep 5
          done
          exit 1

      - name: Rollback frontend on failure
        if: failure()
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: railway redeploy --service frontend --yes


name: Terraform

on:
  pull_request:
    paths: ["infrastructure/**"]
  push:
    branches: [main]
    paths: ["infrastructure/**"]

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        env:
          TF_VAR_railway_token: ${{ secrets.RAILWAY_TOKEN }}
          TF_VAR_stripe_secret_key: ${{ secrets.STRIPE_SECRET_KEY }}
          TF_VAR_stripe_webhook_secret: ${{ secrets.STRIPE_WEBHOOK_SECRET }}
          TF_VAR_stripe_connect_client_id: ${{ secrets.STRIPE_CONNECT_CLIENT_ID }}
          TF_VAR_admin_password: ${{ secrets.ADMIN_PASSWORD }}
          TF_VAR_smtp_password: ${{ secrets.SMTP_PASSWORD }}
          TF_VAR_sentry_dsn: ${{ secrets.SENTRY_DSN }}
          TF_VAR_slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        env:
          TF_VAR_railway_token: ${{ secrets.RAILWAY_TOKEN }}
          TF_VAR_stripe_secret_key: ${{ secrets.STRIPE_SECRET_KEY }}
          TF_VAR_stripe_webhook_secret: ${{ secrets.STRIPE_WEBHOOK_SECRET }}
          TF_VAR_stripe_connect_client_id: ${{ secrets.STRIPE_CONNECT_CLIENT_ID }}
          TF_VAR_admin_password: ${{ secrets.ADMIN_PASSWORD }}
          TF_VAR_smtp_password: ${{ secrets.SMTP_PASSWORD }}
          TF_VAR_sentry_dsn: ${{ secrets.SENTRY_DSN }}
          TF_VAR_slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: terraform apply -auto-approve tfplan


name: Railway Config Drift

on:
  pull_request:
    paths: [".railway/**"]
  push:
    branches: [main]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Railway CLI
        run: npm install -g @railway/cli

      - name: Plan IaC changes
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: |
          railway config plan --out railway-plan.json --detailed-exit-code
        # exit 0 = no changes, exit 2 = changes pending

      - name: Upload plan artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: railway-plan
          path: railway-plan.json


