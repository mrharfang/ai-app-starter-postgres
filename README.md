# AI Starter App with PostgreSQL and Prisma

Opinionated full‑stack skeleton meant to be **forked and cloned** so you (or your AI pair‑programmer) can start coding features immediately instead of scaffolding a project from scratch.

## Repository overview
This monorepo ships a minimal "random fortune" demo to prove everything works end‑to‑end, but the real goal is to provide a ready‑to‑hack stack for rapid AI‑assisted development:

• Backend – an Express + Prisma + PostgreSQL API that returns one random fortune
• Front‑end – a React/Vite SPA that fetches and shows it
• Docker – containerized development environment for PostgreSQL and API
• Azure – deployment configuration for Azure Container Apps and Static Web Apps

## Quick Start
> One-liner productive setup.

```bash
git clone https://github.com/<you>/ai-app-starter-postgres.git
cd ai-app-starter-postgres
npm run bootstrap    # install & generate Prisma client
npm run dev          # DB + API (Docker) & React SPA (host)
```

• SPA → http://localhost:3000  
• API → http://localhost:4001  
• DB  → localhost:5433

## Prerequisites
To run the stack you need:
- **Node.js 20 LTS** (npm is bundled)
- **Docker** (Desktop or Engine)
- **Azure CLI** and **Azure Developer CLI (azd)** for deployment
- **uv** - Modern Python package installer

```bash
# check your versions
node -v   # → v20.x
npm -v    # → 10.x
docker --version # → Docker version 24.x
az --version # → azure-cli x.x.x
azd version # → x.x.x
uv --version # → x.x.x
```

If you need to install or upgrade Node.js:

- **nvm (recommended)**  
  ```bash
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
  nvm install 20
  nvm use 20
  ```

- **Homebrew (macOS)**  
  ```bash
  brew install node@20
  ```

- **Windows** – download the 20 LTS installer from <https://nodejs.org> or use `nvm-windows`.

For Docker, download and install Docker Desktop from https://www.docker.com/products/docker-desktop/

For Azure tools:
```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash  # Linux
brew install azure-cli  # macOS
# Or download from https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

# Install Azure Developer CLI
curl -fsSL https://aka.ms/install-azd.sh | bash  # Linux/macOS
# Or download from https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd
```

> After installing, reopen your terminal so all tools are in PATH.

1. **Fork & clone**
   ```bash
   gh repo create your-new-repo --template jflam/ai-app-starter-postgres --private --clone
   cd your-new-repo
   ```

2. **Install dependencies**

   **Option A (one‑liner)**  
   ```bash
   npm run bootstrap
   ```

   **Option B (manual)**  
   ```bash
   npm install          # root
   cd server && npm install
   cd ../client && npm install
   cd ..
   cd server && npx prisma generate
   ```

3. **Run the dev stack**

   ```bash
   npm run dev
   ```

   • React/Vite SPA on **http://localhost:3000**  
   • Express/Prisma API on **http://localhost:4001** (in Docker)
   • PostgreSQL on **localhost:5433** (in Docker)

## Deploying to Azure

This project includes infrastructure as code and deployment automation for Azure. The deployment process is managed through the Azure Developer CLI (azd) with additional tooling to check quotas and manage configuration.

### Python Dependencies Setup

The deployment tools require Python packages. Set these up using uv:

```bash
# Initialize Python project in the scripts directory
cd scripts
uv init

# Install required packages
uv add uvicorn fastapi pydantic rich aiohttp

# Return to project root
cd ..
```

### Configuration

All Azure resource configurations are centralized in `infra/azure-config.json`. This file defines:

- Required services and their SKUs
- Database configuration
- Allowed deployment regions
- Resource tags and naming conventions

Example configuration:
```json
{
  "required_services": {
    "postgresql": {
      "sku": {
        "name": "Standard_B2s",
        "tier": "Burstable"
      },
      "version": "16",
      "storage_gb": 32
    },
    "container_apps": {
      "resources": {
        "cpu": 0.5,
        "memory": "1Gi"
      }
    }
  }
}
```

### Deployment Steps

1. **Check Azure Quotas and Select Region**

   The quota check script will check available capacity and help you select a deployment region:

   ```bash
   # First, set your subscription ID
   export AZURE_SUBSCRIPTION_ID="your-subscription-id"
   
   # Run the quota check script and select a region
   uv run scripts/check_azure_quota.py
   
   # IMPORTANT: Source the region script in the same terminal
   source ./set_region.sh
   ```

   This will:
   - Check quotas across all configured regions
   - Display available capacity
   - Let you select a region with sufficient quota
   - Create a script to set the deployment region

2. **Configure Azure**

   In the same terminal session:

   ```bash
   # Login to Azure
   az login
   
   # Set default subscription if needed
   az account set --subscription <subscription-id>
   
   # Initialize Azure Developer CLI environment
   azd init
   ```

3. **Set Required Environment Variables**

   ```bash
   azd env set POSTGRES_ADMIN_PASSWORD "your-secure-password"
   ```

4. **Deploy**

   ```bash
   # Deploy using the selected region
   azd up
   ```

   The deployment process will:
   - Generate Bicep parameters from azure-config.json
   - Create or update Azure resources in the selected region
   - Build and deploy the application
   - Configure all necessary connections

### Deployment Architecture

The application deploys the following Azure resources:

- **Azure Container Apps** - Hosts the API
- **Azure Database for PostgreSQL Flexible Server** - Database
- **Azure Container Registry** - Stores Docker images
- **Azure Static Web Apps** - Hosts the front-end
- **Log Analytics Workspace** - Centralized logging

### Post-Deployment

After deployment completes:

1. **Run Database Migrations**
   ```bash
   cd server
   DATABASE_URL="<connection-string>" npx prisma migrate deploy
   ```

2. **Seed the Database**
   ```bash
   DATABASE_URL="<connection-string>" npx prisma db seed
   ```

3. **Verify the Deployment**
   ```bash
   # Get the deployed endpoints
   azd show
   ```

### Troubleshooting

- If deployment fails due to quota issues, run the quota check script and either:
  - Choose a different region with sufficient quota
  - Request a quota increase through Azure Portal
- For database connection issues, verify the firewall rules are correctly configured
- Check Container Apps logs in Azure Portal for API issues
- For Static Web App issues, verify the API URL environment variable is correctly set

## Building for production

### 1. Compile the backend (server)

The backend is containerized and can be built with:

```bash
docker compose build api
```

Or running directly:

```bash
cd server
npm install        # first‑time only
npm run build      # emits JS to server/dist/
NODE_ENV=production node dist/index.js
```

### 2. Build the front‑end (client)

```bash
cd client
npm install        # first‑time only
npm run build      # outputs static assets to client/dist/
```

### 3. Serve the SPA

Any static host (Nginx, Netlify, Vercel, S3, etc.) can serve the `client/dist/` folder.

Local preview:

```bash
cd client
npm run preview    # opens http://localhost:4173
```

### 4. One‑shot helper from the repo root

```bash
npm run bootstrap                     # ensure all deps
docker compose build api \
  && (cd client && npm run build)
```

## Environment variables

- `PORT` - API port (default 4000 inside the container)
- `DATABASE_URL` - PostgreSQL connection string
- `AZURE_SUBSCRIPTION_ID` - Required for quota checks and deployment

## Docker Compose configuration

The `docker-compose.yml` defines:

- `db` service - PostgreSQL 16 Alpine with credentials in the compose file
- `api` service - Node.js API with Prisma connecting to the database

To run only the database:

```bash
docker compose up -d db
```

To rebuild and run the API:

```bash
docker compose up --build api
```

To run everything:

```bash
docker compose up
```

## Deploy to Azure  

This starter app comes pre-configured for deployment to Azure using the Azure Developer CLI (azd). The deployment architecture consists of:

- **Azure Container App** - Hosts the Express API backend
- **Azure PostgreSQL Flexible Server** - Managed PostgreSQL database
- **Azure Static Web App** - Hosts the React frontend
- **Azure Container Registry** - Stores the Docker image for the API

### Prerequisites for Azure Deployment

1. **Install Azure CLI and Azure Developer CLI (azd)**
   ```bash
   # Install Azure CLI
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash  # Linux
   brew install azure-cli                                   # macOS
   winget install -e --id Microsoft.AzureCLI               # Windows
   
   # Install Azure Developer CLI
   curl -fsSL https://aka.ms/install-azd.sh | bash         # Linux/macOS
   winget install -e --id Microsoft.Azd                    # Windows
   ```

2. **Login to Azure**
   ```bash
   az login
   azd auth login
   ```

### Deploying the Application

1. **Initialize the Azure environment**
   ```bash
   azd env new myenv
   ```
   Replace `myenv` with your preferred environment name (e.g., dev, staging, prod).

2. **Set database password**
   ```bash
   azd env set POSTGRES_ADMIN_PASSWORD "<your-secure-password>"
   ```
   Make sure to use a strong password that meets Azure requirements (at least 8 characters with lowercase, uppercase, numbers, and symbols).

3. **Provision resources and deploy the application**
   ```bash
   azd up
   ```
   This command provisions all Azure resources and deploys both the API and frontend. It may take several minutes to complete.

4. **Verify the deployed application**
   After successful deployment, you'll see URLs for both services:
   - Frontend (Static Web App): `https://<random-name>.<hash>.azurestaticapps.net`
   - Backend API (Container App): `https://server.<unique-id>.<region>.azurecontainerapps.io`

### Environment Configuration

#### Frontend Configuration

The frontend application needs to know the URL of the backend API. There are two ways to configure this:

1. **Runtime Configuration (Option 1) - Recommended**

   Configure application settings in Azure Static Web Apps:
   ```bash
   az staticwebapp appsettings set \
     --name <your-static-webapp-name> \
     --resource-group <resource-group-name> \
     --setting-names VITE_API_BASE_URL="https://server.<unique-id>.<region>.azurecontainerapps.io"
   ```

2. **Build-time Configuration (Option 2)**

   Update the `.env.production` file before building:
   ```
   VITE_API_BASE_URL=https://server.<unique-id>.<region>.azurecontainerapps.io
   ```

#### Database Migrations

Azure Container Apps automatically applies Prisma migrations during startup. If you need to run migrations manually:

```bash
# Get the connection string from Azure
DATABASE_URL=$(az containerapp secret show \
  --name server \
  --resource-group <resource-group-name> \
  --secret-name database-url \
  --query value -o tsv)

# Run migrations
cd server
npm run migrate
```

### CI/CD Pipeline Setup

You can set up a GitHub Actions workflow for continuous deployment:

1. **Configure the GitHub Actions pipeline**
   ```bash
   azd pipeline config
   ```
   This command will:
   - Create a service principal for GitHub
   - Set up the necessary GitHub repository secrets
   - Generate a workflow file in `.github/workflows/azure-dev.yml`

2. **Push changes to trigger deployment**
   After the pipeline is configured, any push to the main branch will trigger a deployment to your Azure environment.

### Iterative Development

For local development connected to Azure resources:

1. **Get the Azure database connection string**
   ```bash
   az postgres flexible-server show-connection-string \
     --server-name <postgres-server-name> \
     --database-name <database-name> \
     --admin-user <admin-username> \
     --query connectionString
   ```

2. **Add your IP to the PostgreSQL firewall rules**
   ```bash
   az postgres flexible-server firewall-rule create \
     --resource-group <resource-group-name> \
     --name <postgres-server-name> \
     --rule-name AllowMyIP \
     --start-ip-address <your-ip-address> \
     --end-ip-address <your-ip-address>
   ```

3. **Set environment variables for local development**
   Create a `.env.local` file in the client directory:
   ```
   VITE_API_BASE_URL=http://localhost:4000
   ```

   Create a `.env` file in the server directory:
   ```
   DATABASE_URL=<azure-postgres-connection-string>
   ```

### Troubleshooting Azure Deployment

If your deployment fails or the application isn't working properly:

1. **Check Container App logs**
   ```bash
   az containerapp logs show \
     --name server \
     --resource-group <resource-group-name> \
     --follow
   ```

2. **Check Static Web App deployment status**
   ```bash
   az staticwebapp show \
     --name <static-webapp-name> \
     --resource-group <resource-group-name>
   ```

3. **Common issues**:
   - **CORS errors**: Ensure the Container App's CORS policy allows requests from the Static Web App domain
   - **Database connection issues**: Check if the database URL secret is correctly set in the Container App
   - **Environment variable configuration**: Verify that `VITE_API_BASE_URL` is correctly set in the Static Web App

4. **Clean up resources**
   If you need to remove all deployed resources:
   ```bash
   azd down
   ```

## Security Considerations

- Environment variables prefixed with `VITE_` are embedded in the client-side code and visible to users
- Only include non-sensitive information in client-side environment variables
- Sensitive data like database credentials are securely stored as Container App secrets
- The Static Web App's Content Security Policy (CSP) in `staticwebapp.config.json` restricts which domains can be connected to
