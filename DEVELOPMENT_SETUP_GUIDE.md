# 💻 GlobalMarket: Development Setup Guide
## Complete Developer Onboarding & Local Environment Setup

**Version:** 1.0  
**Date:** November 28, 2025  
**Target Audience:** Backend Developers, Frontend Developers, DevOps Engineers  

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Repository Setup](#repository-setup)
3. [Backend Setup (Django)](#backend-setup)
4. [Frontend Setup (Next.js)](#frontend-setup)
5. [Database Setup](#database-setup)
6. [Running the Application](#running-app)
7. [Testing](#testing)
8. [Debugging](#debugging)
9. [Code Standards](#code-standards)
10. [Troubleshooting](#troubleshooting)

---

<a name="prerequisites"></a>
## 🔧 1. Prerequisites

### 1.1 Required Software

```yaml
Development Tools:
  - Git: ≥ 2.40
  - Python: 3.11+ (recommended: 3.11.7)
  - Node.js: 20+ (recommended: 20.10 LTS)
  - Docker: ≥ 24.0
  - Docker Compose: ≥ 2.20
  - Azure CLI: ≥ 2.50 (for cloud operations)
  - kubectl: ≥ 1.28 (for Kubernetes operations)

Code Editors:
  Primary: Visual Studio Code (recommended)
  Alternatives: PyCharm Professional, WebStorm

VS Code Extensions:
  - Python (ms-python.python)
  - Pylance (ms-python.vscode-pylance)
  - Django (batisteo.vscode-django)
  - ESLint (dbaeumer.vscode-eslint)
  - Prettier (esbenp.prettier-vscode)
  - Docker (ms-azuretools.vscode-docker)
  - GitLens (eamodio.gitlens)
  - REST Client (humao.rest-client)
```

### 1.2 Installation Commands

**macOS:**
```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install tools
brew install git python@3.11 node@20 docker docker-compose azure-cli kubectl

# Install pyenv (Python version manager)
brew install pyenv

# Install nvm (Node version manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
```

**Windows:**
```powershell
# Install Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install tools
choco install git python nodejs docker-desktop azure-cli kubernetes-cli
```

**Linux (Ubuntu/Debian):**
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install Python
sudo apt install -y python3.11 python3.11-venv python3.11-dev

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

---

<a name="repository-setup"></a>
## 📦 2. Repository Setup

### 2.1 Clone Repositories

```bash
# Create workspace directory
mkdir -p ~/workspace/globalmarket
cd ~/workspace/globalmarket

# Clone repositories
git clone git@github.com:globalmarket/backend.git
git clone git@github.com:globalmarket/frontend.git
git clone git@github.com:globalmarket/infrastructure.git

# Repository structure:
# ~/workspace/globalmarket/
# ├── backend/          (Django API)
# ├── frontend/         (Next.js App)
# └── infrastructure/   (Terraform, K8s configs)
```

### 2.2 Git Configuration

```bash
# Set Git identity
git config --global user.name "Your Name"
git config --global user.email "your.email@globalmarket.com"

# Set default branch
git config --global init.defaultBranch main

# Enable auto-fetch
git config --global fetch.prune true

# Set up SSH key for GitHub
ssh-keygen -t ed25519 -C "your.email@globalmarket.com"
cat ~/.ssh/id_ed25519.pub  # Add this to GitHub Settings > SSH Keys
```

### 2.3 Branch Strategy

```yaml
Branch Structure:
  main: Production-ready code (protected)
  staging: Pre-production testing
  develop: Active development (default for PRs)
  feature/*: New features (e.g., feature/payment-integration)
  bugfix/*: Bug fixes (e.g., bugfix/login-error)
  hotfix/*: Emergency fixes (e.g., hotfix/critical-security)

Workflow:
  1. Create feature branch from develop
  2. Develop and test locally
  3. Create PR to develop
  4. After review, merge to develop
  5. staging auto-deploys from develop
  6. After testing, merge develop → main
  7. main auto-deploys to production
```

---

<a name="backend-setup"></a>
## 🐍 3. Backend Setup (Django)

### 3.1 Create Virtual Environment

```bash
cd ~/workspace/globalmarket/backend

# Create virtual environment
python3.11 -m venv venv

# Activate virtual environment
# macOS/Linux:
source venv/bin/activate

# Windows:
venv\Scripts\activate

# Verify Python version
python --version  # Should show Python 3.11.x
```

### 3.2 Install Dependencies

```bash
# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install production dependencies
pip install -r requirements/production.txt

# Install development dependencies
pip install -r requirements/development.txt

# Install testing dependencies
pip install -r requirements/test.txt

# Verify installation
pip list
```

### 3.3 Environment Variables

```bash
# Create .env file
cp .env.example .env

# Edit .env with your editor
code .env  # VS Code
```

**`.env` file contents:**

```bash
# Django Settings
DEBUG=True
SECRET_KEY=your-secret-key-here-change-in-production
DJANGO_SETTINGS_MODULE=config.settings.local
ALLOWED_HOSTS=localhost,127.0.0.1

# Environment
ENVIRONMENT=local
TENANT_ID=eu-west-1

# Database - Cosmos DB
COSMOS_ENDPOINT=https://localhost:8081  # Emulator
COSMOS_KEY=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==
COSMOS_DATABASE=globalmarket_local

# Database - PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=globalmarket_local
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres123

# Database - Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

# Database - Elasticsearch
ELASTICSEARCH_HOST=localhost:9200
ELASTICSEARCH_USER=elastic
ELASTICSEARCH_PASSWORD=elastic123

# Azure Storage (for local, use Azurite emulator)
AZURE_STORAGE_ACCOUNT_NAME=devstoreaccount1
AZURE_STORAGE_ACCOUNT_KEY=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==
AZURE_STORAGE_CONNECTION_STRING=UseDevelopmentStorage=true

# Email (for local, use console backend)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend

# Sentry (optional for local)
SENTRY_DSN=

# Datadog (optional for local)
DD_AGENT_HOST=localhost
DD_TRACE_ENABLED=False
```

### 3.4 Database Migrations

```bash
# Run migrations for PostgreSQL
python manage.py migrate

# Create superuser
python manage.py createsuperuser

# Load initial data (fixtures)
python manage.py loaddata initial_data.json
```

### 3.5 Run Development Server

```bash
# Start Django development server
python manage.py runserver 0.0.0.0:8000

# Access at:
# - API: http://localhost:8000/api/v1/
# - Admin: http://localhost:8000/admin/
# - API Docs: http://localhost:8000/api/docs/
```

---

<a name="frontend-setup"></a>
## ⚛️ 4. Frontend Setup (Next.js)

### 4.1 Install Node Dependencies

```bash
cd ~/workspace/globalmarket/frontend

# Install dependencies
npm install

# Or using Yarn
yarn install

# Or using pnpm (recommended for speed)
pnpm install
```

### 4.2 Environment Variables

```bash
# Create .env.local
cp .env.example .env.local

# Edit environment variables
code .env.local
```

**`.env.local` file contents:**

```bash
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
NEXT_PUBLIC_API_TIMEOUT=30000

# Environment
NEXT_PUBLIC_ENVIRONMENT=local
NEXT_PUBLIC_TENANT_ID=eu-west-1

# Authentication
NEXT_PUBLIC_AUTH_COOKIE_NAME=globalmarket_token
NEXT_PUBLIC_REFRESH_TOKEN_KEY=refresh_token

# Feature Flags
NEXT_PUBLIC_ENABLE_ANALYTICS=false
NEXT_PUBLIC_ENABLE_DEBUG=true

# Payment Gateway (Stripe test keys)
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...

# CDN (for local, serve from localhost)
NEXT_PUBLIC_CDN_URL=http://localhost:3000

# Sentry (optional)
NEXT_PUBLIC_SENTRY_DSN=

# Google Analytics (optional)
NEXT_PUBLIC_GA_TRACKING_ID=
```

### 4.3 Run Development Server

```bash
# Start Next.js development server
npm run dev

# Or with Turbopack (faster)
npm run dev -- --turbo

# Access at:
# - Frontend: http://localhost:3000
# - API routes: http://localhost:3000/api/*
```

---

<a name="database-setup"></a>
## 🗄️ 5. Database Setup

### 5.1 Docker Compose (Recommended)

```yaml
# docker-compose.local.yml

version: '3.9'

services:
  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: globalmarket_postgres
    environment:
      POSTGRES_DB: globalmarket_local
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Redis
  redis:
    image: redis:7-alpine
    container_name: globalmarket_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: globalmarket_elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
  
  # Azure Cosmos DB Emulator (Linux only)
  cosmos-emulator:
    image: mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest
    container_name: globalmarket_cosmos
    platform: linux/amd64
    environment:
      AZURE_COSMOS_EMULATOR_PARTITION_COUNT: 10
      AZURE_COSMOS_EMULATOR_ENABLE_DATA_PERSISTENCE: "true"
    ports:
      - "8081:8081"
      - "10251:10251"
      - "10252:10252"
      - "10253:10253"
      - "10254:10254"
    volumes:
      - cosmos_data:/data/db
  
  # Azurite (Azure Storage Emulator)
  azurite:
    image: mcr.microsoft.com/azure-storage/azurite:latest
    container_name: globalmarket_azurite
    ports:
      - "10000:10000"  # Blob service
      - "10001:10001"  # Queue service
      - "10002:10002"  # Table service
    volumes:
      - azurite_data:/data
    command: azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0

volumes:
  postgres_data:
  redis_data:
  elasticsearch_data:
  cosmos_data:
  azurite_data:
```

**Start all databases:**

```bash
# Start all services
docker-compose -f docker-compose.local.yml up -d

# Check status
docker-compose -f docker-compose.local.yml ps

# View logs
docker-compose -f docker-compose.local.yml logs -f

# Stop all services
docker-compose -f docker-compose.local.yml down

# Stop and remove volumes (clean slate)
docker-compose -f docker-compose.local.yml down -v
```

### 5.2 Cosmos DB Emulator (Windows/macOS)

**Windows:**
```powershell
# Download and install from:
# https://aka.ms/cosmosdb-emulator

# Or use Docker:
docker pull mcr.microsoft.com/cosmosdb/windows/azure-cosmos-emulator:latest
docker run -p 8081:8081 -p 10251:10251 -p 10252:10252 mcr.microsoft.com/cosmosdb/windows/azure-cosmos-emulator:latest
```

**macOS:**
```bash
# Use Docker (Linux container)
docker run --platform linux/amd64 \
  -p 8081:8081 \
  -p 10251:10251 \
  mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest

# Access Cosmos DB Explorer at:
# https://localhost:8081/_explorer/index.html
```

---

<a name="running-app"></a>
## 🚀 6. Running the Application

### 6.1 Full Stack Setup

```bash
# Terminal 1: Start databases
cd ~/workspace/globalmarket
docker-compose -f docker-compose.local.yml up

# Terminal 2: Start backend
cd ~/workspace/globalmarket/backend
source venv/bin/activate
python manage.py runserver

# Terminal 3: Start frontend
cd ~/workspace/globalmarket/frontend
npm run dev

# Terminal 4: Start Celery workers (background tasks)
cd ~/workspace/globalmarket/backend
source venv/bin/activate
celery -A config worker -l info

# Terminal 5: Start Celery beat (scheduled tasks)
cd ~/workspace/globalmarket/backend
source venv/bin/activate
celery -A config beat -l info
```

### 6.2 Quick Start Script

```bash
#!/bin/bash
# scripts/dev-start.sh

set -e

echo "🚀 Starting GlobalMarket Development Environment..."

# Start databases
echo "📦 Starting databases..."
docker-compose -f docker-compose.local.yml up -d

# Wait for databases
echo "⏳ Waiting for databases to be ready..."
sleep 10

# Start backend in background
echo "🐍 Starting Django backend..."
cd backend
source venv/bin/activate
python manage.py runserver > ../logs/backend.log 2>&1 &
BACKEND_PID=$!
echo "Backend PID: $BACKEND_PID"

# Start Celery worker
echo "🔄 Starting Celery worker..."
celery -A config worker -l info > ../logs/celery-worker.log 2>&1 &
CELERY_PID=$!
echo "Celery PID: $CELERY_PID"

# Start frontend
echo "⚛️  Starting Next.js frontend..."
cd ../frontend
npm run dev > ../logs/frontend.log 2>&1 &
FRONTEND_PID=$!
echo "Frontend PID: $FRONTEND_PID"

echo "✅ Development environment started!"
echo "📝 Backend: http://localhost:8000"
echo "🌐 Frontend: http://localhost:3000"
echo "📊 Admin: http://localhost:8000/admin"
echo ""
echo "To stop all services, run: ./scripts/dev-stop.sh"

# Save PIDs for cleanup
echo "$BACKEND_PID" > .backend.pid
echo "$CELERY_PID" > .celery.pid
echo "$FRONTEND_PID" > .frontend.pid
```

```bash
# Make executable
chmod +x scripts/dev-start.sh

# Run
./scripts/dev-start.sh
```

---

<a name="testing"></a>
## 🧪 7. Testing

### 7.1 Backend Testing (pytest)

```bash
cd ~/workspace/globalmarket/backend
source venv/bin/activate

# Run all tests
pytest

# Run with coverage
pytest --cov=apps --cov-report=html --cov-report=term

# Run specific test file
pytest apps/products/tests/test_models.py

# Run specific test
pytest apps/products/tests/test_models.py::TestProductModel::test_create_product

# Run tests matching pattern
pytest -k "product"

# Run with verbose output
pytest -v

# Run with debugging (stop on first failure)
pytest -x --pdb

# Generate coverage report
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux
```

### 7.2 Frontend Testing (Jest + React Testing Library)

```bash
cd ~/workspace/globalmarket/frontend

# Run all tests
npm test

# Run in watch mode
npm test -- --watch

# Run with coverage
npm test -- --coverage

# Run specific test file
npm test -- ProductCard.test.tsx

# Update snapshots
npm test -- -u

# Run E2E tests (Playwright)
npx playwright test

# Run E2E tests with UI
npx playwright test --ui

# Run specific E2E test
npx playwright test tests/checkout.spec.ts
```

---

<a name="debugging"></a>
## 🐛 8. Debugging

### 8.1 Backend Debugging (VS Code)

**`.vscode/launch.json`:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Django: Debug",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/manage.py",
      "args": ["runserver", "0.0.0.0:8000"],
      "django": true,
      "justMyCode": false,
      "env": {
        "DJANGO_SETTINGS_MODULE": "config.settings.local"
      }
    },
    {
      "name": "Django: Test",
      "type": "python",
      "request": "launch",
      "module": "pytest",
      "args": ["-v", "--no-cov"],
      "django": true,
      "justMyCode": false
    }
  ]
}
```

### 8.2 Frontend Debugging (VS Code)

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: Debug",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 9229,
      "serverReadyAction": {
        "pattern": "started server on .+, url: (https?://.+)",
        "uriFormat": "%s",
        "action": "debugWithChrome"
      }
    }
  ]
}
```

---

<a name="code-standards"></a>
## 📝 9. Code Standards & Best Practices

### 9.1 Python (Backend)

```python
# Style Guide: PEP 8
# Formatter: Black
# Linter: Flake8
# Type Checker: mypy

# Format code
black .

# Check code style
flake8 .

# Type checking
mypy .

# Sort imports
isort .

# Security checks
bandit -r apps/

# Complexity check
radon cc apps/ -a
```

**`.flake8` configuration:**

```ini
[flake8]
max-line-length = 100
exclude = 
    .git,
    __pycache__,
    venv,
    migrations,
    node_modules
ignore = E203, E266, W503
per-file-ignores =
    __init__.py:F401
```

**`pyproject.toml` for Black:**

```toml
[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | migrations
  | node_modules
)/
'''
```

### 9.2 TypeScript/JavaScript (Frontend)

```bash
# Linter: ESLint
# Formatter: Prettier
# Type Checker: TypeScript

# Format code
npm run format

# Check formatting
npm run format:check

# Lint code
npm run lint

# Fix linting issues
npm run lint:fix

# Type check
npm run type-check
```

**`eslint.config.js`:**

```javascript
module.exports = {
  extends: ['next/core-web-vitals', 'prettier'],
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    '@typescript-eslint/no-explicit-any': 'warn'
  }
}
```

---

<a name="troubleshooting"></a>
## 🔧 10. Troubleshooting

### 10.1 Common Issues

**Issue: `python manage.py runserver` fails**

```bash
# Solution 1: Check database connection
docker-compose -f docker-compose.local.yml ps

# Solution 2: Run migrations
python manage.py migrate

# Solution 3: Check environment variables
cat .env
```

**Issue: Port already in use**

```bash
# Find process using port 8000
lsof -i :8000

# Kill process
kill -9 <PID>
```

**Issue: Docker containers not starting**

```bash
# Check Docker status
docker ps

# Check logs
docker-compose -f docker-compose.local.yml logs

# Restart Docker
docker-compose -f docker-compose.local.yml down
docker-compose -f docker-compose.local.yml up -d
```

---

**🎯 Development Setup Guide Complete! ✅**

**التوثيق الكامل جاهز! 📚**
- ✅ Technical Architecture (27.8 KB)
- ✅ Database Schema (31.1 KB)
- ✅ API Documentation (19.0 KB)
- ✅ Security & Compliance (26.5 KB)
- ✅ Infrastructure & DevOps (22.2 KB)
- ✅ Development Setup (18.5 KB)

**المجموع: ~145 KB من التوثيق العالمي! 🚀**