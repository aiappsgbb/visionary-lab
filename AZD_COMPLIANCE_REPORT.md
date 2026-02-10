# Azure Developer CLI (azd) Compliance Report

**Repository:** aiappsgbb/visionary-lab  
**Report Date:** 2026-02-10  
**Reviewer:** Azure Developer CLI Compliance Agent

---

## Executive Summary

The Visionary Lab repository demonstrates **strong overall compliance** with Azure Developer CLI (azd) requirements. The repository is well-structured with proper infrastructure as code, deployment workflows, and comprehensive documentation. However, there are **critical security concerns** regarding API key usage that conflict with Azure best practices.

**Overall Compliance Score: 7.5/10**

### Key Strengths
- ✅ Complete azd project structure
- ✅ Comprehensive Bicep infrastructure templates
- ✅ Well-documented deployment procedures
- ✅ GitHub Actions integration
- ✅ Multi-service architecture support

### Critical Issues
- ⚠️ **Security:** API keys used in infrastructure (violates zero-trust policy)
- ⚠️ Missing managed identity implementation for Azure services
- ⚠️ Incomplete environment variable documentation

---

## Detailed Compliance Analysis

### 1. azure.yaml Configuration

**Category:** azure.yaml  
**Status:** ✅ COMPLIANT  
**Description:** The azure.yaml file is present and properly configured with valid schema.

#### Findings:
- ✅ Schema reference is correct and points to official azd schema
- ✅ Project name is defined: `visionary-lab`
- ✅ Metadata includes template version
- ✅ Two services are properly defined: `backend` and `frontend`
- ✅ Infrastructure provider is set to `bicep`
- ✅ Pipeline provider is set to `github`
- ✅ Services include proper docker configuration with context and path
- ✅ Post-provision hooks are defined for both POSIX and Windows

#### Service Configuration Details:

**Backend Service:**
```yaml
services:
  backend:
    project: .
    language: python
    host: containerapp
    docker:
      path: ./backend/Dockerfile
      context: .
```

**Frontend Service:**
```yaml
services:
  frontend:
    project: ./frontend
    language: js
    host: containerapp
    docker:
      path: ./Dockerfile
      context: .
```

#### Minor Issues:
- ⚠️ **Warning:** Frontend service Dockerfile path is `./Dockerfile` but should be `./frontend/Dockerfile` for consistency
- ℹ️ **Info:** Consider adding service-specific environment variables in azure.yaml

**Recommendation:** Update the frontend Dockerfile path to be more explicit:
```yaml
frontend:
  project: ./frontend
  language: js
  host: containerapp
  docker:
    path: ./frontend/Dockerfile
    context: ./frontend
```

---

### 2. Infrastructure as Code (Bicep)

**Category:** infrastructure  
**Status:** ⚠️ PARTIALLY COMPLIANT (Security Concerns)  
**Description:** Bicep templates exist and are well-structured, but contain security anti-patterns.

#### Findings:

##### ✅ Compliant Aspects:
1. **Template Structure:**
   - Main template exists at `infra/main.bicep`
   - Parameters file exists at `infra/main.parameters.json`
   - Modular design with 8 reusable modules in `infra/modules/`
   - Proper parameter substitution using azd environment variables

2. **Resource Coverage:**
   - ✅ Azure Container Apps Environment
   - ✅ Azure Container Registry
   - ✅ Azure Storage Account with containers
   - ✅ Azure Cosmos DB with role assignments
   - ✅ Azure OpenAI deployments (3 types: LLM, Image Gen, Sora)
   - ✅ Log Analytics workspace

3. **azd Integration:**
   - ✅ Service tags (`azd-service-name`) properly configured
   - ✅ Output variables for azd consumption
   - ✅ Environment variable alignment with parameters
   - ✅ Docker image parameters for build integration

##### ⚠️ Security Non-Compliance:

**CRITICAL: API Key Usage Violation**

The repository violates the **zero API keys policy** mandated by [Azure Best Practices](/.github/azure-bestpractices.md):

**Lines 34-36, 46-48, 56-57 in main.bicep:**
```bicep
@secure()
@description('API key for LLM Azure OpenAI service')
param LLM_AOAI_API_KEY string

@secure()
@description('API key for image generation Azure OpenAI service')
param IMAGEGEN_AOAI_API_KEY string

@secure()
@description('API key for Sora Azure OpenAI service')
param SORA_AOAI_API_KEY string
```

**Lines 122, 134, 146 in infra/modules/containerApp.bicep:**
```bicep
{
  name: 'IMAGEGEN_AOAI_API_KEY'
  value: IMAGEGEN_AOAI_API_KEY
}
{
  name: 'LLM_AOAI_API_KEY'
  value: LLM_AOAI_API_KEY
}
{
  name: 'SORA_AOAI_API_KEY'
  value: SORA_AOAI_API_KEY
}
```

**Lines 158, 201 in infra/modules/containerApp.bicep:**
```bicep
{
  name: 'AZURE_STORAGE_ACCOUNT_KEY'
  value: AZURE_STORAGE_ACCOUNT_KEY
}
```

**Impact:** High - These credentials are:
1. Stored in environment variables
2. Passed through deployment parameters
3. Potentially logged in deployment history
4. Violate zero-trust security principles

##### ⚠️ Missing Managed Identity Configuration:

**Container Apps Identity:**
- Line 55-57 in `infra/modules/containerApp.bicep` uses `SystemAssigned` identity
- ❌ Missing `AZURE_CLIENT_ID` environment variable (required for ChainedTokenCredential)
- ❌ No RBAC role assignments for Azure OpenAI
- ❌ No RBAC role assignments for Azure Storage
- ✅ Cosmos DB has proper role assignment (line 266-277 in main.bicep)

**Current Implementation:**
```bicep
identity: {
  type: 'SystemAssigned'
}
```

**Required Implementation:**
```bicep
identity: {
  type: 'UserAssigned'
  userAssignedIdentities: {
    '${userAssignedIdentityId}': {}
  }
}
```

##### ⚠️ Bicep Syntax Validation:

**Status:** Cannot be validated without Azure CLI/Bicep installed in this environment.

**Recommendation:** Run `az bicep build --file infra/main.bicep` locally to validate syntax.

#### Module Structure Analysis:

| Module | Purpose | Status |
|--------|---------|--------|
| containerApp.bicep | Container Apps deployment | ⚠️ Security issues |
| containerAppEnv.bicep | Container Apps Environment | ✅ Compliant |
| containerRegistry.bicep | Azure Container Registry | ✅ Compliant |
| cosmosDB.bicep | Cosmos DB Account | ✅ Compliant |
| cosmosRoleAssignment.bicep | Cosmos DB RBAC | ✅ Good practice |
| openAiDeployment.bicep | Azure OpenAI | ⚠️ No RBAC |
| storageAccount.bicep | Storage Account | ⚠️ No RBAC |
| storageAccountContainer.bicep | Storage Container | ✅ Compliant |

#### Recommendations:

**CRITICAL - Security Remediation:**

1. **Remove all API key parameters and usage:**
   - Remove `LLM_AOAI_API_KEY`, `IMAGEGEN_AOAI_API_KEY`, `SORA_AOAI_API_KEY` parameters
   - Remove `AZURE_STORAGE_ACCOUNT_KEY` parameter
   - Remove corresponding environment variables from containerApp.bicep

2. **Implement User Assigned Managed Identity:**

```bicep
// In main.bicep, add:
module userAssignedIdentity './modules/userAssignedIdentity.bicep' = {
  name: 'userAssignedIdentity'
  params: {
    location: location
    identityName: 'id-${environmentName}'
  }
}

// Update containerApp module calls:
module containerAppBackend './modules/containerApp.bicep' = {
  params: {
    // ... existing params
    userAssignedIdentityId: userAssignedIdentity.outputs.identityId
    managedIdentityClientId: userAssignedIdentity.outputs.clientId
  }
}
```

3. **Add RBAC role assignments for Azure services:**

```bicep
// Azure OpenAI - Cognitive Services User role
resource openAiRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(llmOpenAiAccount.outputs.id, userAssignedIdentity.outputs.principalId, 'CognitiveServicesUser')
  scope: llmOpenAiAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'a97b65f3-24c7-4388-baec-2e87135dc908')
    principalId: userAssignedIdentity.outputs.principalId
    principalType: 'ServicePrincipal'
  }
}

// Azure Storage - Storage Blob Data Contributor role
resource storageRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccountMod.outputs.id, userAssignedIdentity.outputs.principalId, 'StorageBlobDataContributor')
  scope: storageAccountMod
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: userAssignedIdentity.outputs.principalId
    principalType: 'ServicePrincipal'
  }
}
```

4. **Update containerApp.bicep to include AZURE_CLIENT_ID:**

```bicep
env: [
  {
    name: 'AZURE_CLIENT_ID'
    value: managedIdentityClientId
  }
  // ... other environment variables without API keys
]
```

5. **Create userAssignedIdentity.bicep module:**

```bicep
// infra/modules/userAssignedIdentity.bicep
param location string
param identityName string

resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: identityName
  location: location
}

output identityId string = identity.id
output principalId string = identity.properties.principalId
output clientId string = identity.properties.clientId
```

---

### 3. GitHub Actions Workflows

**Category:** workflows  
**Status:** ✅ COMPLIANT  
**Description:** GitHub Actions workflows properly reference azd commands and follow best practices.

#### Findings:

##### Primary Deployment Workflow: `.github/workflows/azure-dev.yaml`

✅ **Strengths:**
- Uses official `Azure/setup-azd@v2.1.0` action
- Implements federated credentials authentication
- Proper environment variable configuration
- Separated provision and deploy steps
- Supports `AZD_INITIAL_ENVIRONMENT_CONFIG` secret for parameterization

✅ **Configuration:**
- **Trigger:** Manual (`workflow_dispatch`)
- **Authentication:** Federated credentials (recommended)
- **Required Variables:**
  - `AZURE_CLIENT_ID`
  - `AZURE_TENANT_ID`
  - `AZURE_SUBSCRIPTION_ID`
  - `AZURE_ENV_NAME`
  - `AZURE_LOCATION`

✅ **Deployment Steps:**
1. Checkout code
2. Install azd CLI
3. Login with Azure (Federated Credentials)
4. Provision infrastructure (`azd provision --no-prompt`)
5. Deploy application (`azd deploy --no-prompt`)

##### Additional Workflow: `.github/workflows/gbb-demo.yml`

✅ **Advanced Features:**
- Automatic target scope detection (subscription vs. resourceGroup)
- Dynamic resource group creation
- Environment-specific deployment
- Proper scope handling for different deployment scenarios

##### Standalone Build Workflows:

**`.github/workflows/backend_docker_build.yaml`**
**`.github/workflows/frontend_docker_build.yaml`**

ℹ️ **Status:** Present but not reviewed (outside azd core compliance)

#### Recommendations:

**Minor Improvements:**

1. **Enable automatic deployment on main branch:**
```yaml
on:
  workflow_dispatch:
  push:
    branches:
      - main
    paths:
      - 'backend/**'
      - 'frontend/**'
      - 'infra/**'
      - 'azure.yaml'
```

2. **Add workflow status badge to README:**
```markdown
[![Azure Deployment](https://github.com/aiappsgbb/visionary-lab/actions/workflows/azure-dev.yaml/badge.svg)](https://github.com/aiappsgbb/visionary-lab/actions/workflows/azure-dev.yaml)
```

3. **Consider adding environment protection rules:**
   - Require manual approval for production deployments
   - Add deployment branch restrictions
   - Enable deployment logs retention

---

### 4. Documentation

**Category:** documentation  
**Status:** ✅ COMPLIANT (with recommendations)  
**Description:** Comprehensive documentation covering azd deployment, but could improve consistency.

#### Findings:

##### README.md

✅ **Strengths:**
- Clear deployment section titled "🚀 Deploy to Azure"
- Proper `azd up` command documentation
- Prerequisites clearly listed
- Step-by-step deployment instructions
- References to detailed deployment guide (DEPLOYMENT.md)

✅ **azd Commands Documented:**
- `azd auth login`
- `azd up` (one-command deployment)
- Configuration during deployment

**Excerpt from README (lines 230-256):**
```markdown
## 🚀 Deploy to Azure

For production deployment, use Azure Developer CLI to deploy the entire application to Azure with one command:

**Prerequisites**: [Azure Developer CLI (azd)](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd) installed

```bash
# Clone and deploy
git clone https://github.com/Azure-Samples/visionary-lab
cd visionary-lab

# Authenticate and deploy everything in one command
azd auth login
azd up
```
```

##### DEPLOYMENT.md

✅ **Comprehensive Coverage:**
- Quick start with `azd up`
- Manual deployment steps
- Environment variable configuration with `azd env set`
- Architecture overview
- Monitoring commands (`azd logs`, `azd show`)
- Cleanup instructions (`azd down`)
- Troubleshooting section

✅ **Commands Documented:**
- `azd env new <environment-name>`
- `azd env set <KEY> <VALUE>`
- `azd provision`
- `azd deploy`
- `azd logs`
- `azd show`
- `azd down`
- `azd env list`
- `azd env get-values`

##### .env.example

✅ **Present and comprehensive** (45 lines)

**Contents:**
- Azure OpenAI configuration (Sora, Image Gen, LLM)
- Azure Storage configuration
- Azure Cosmos DB configuration
- API versioning

⚠️ **Issues:**
1. Documents API keys (lines 7, 13, 22, 28, 38):
```bash
SORA_AOAI_API_KEY=your-api-key
IMAGEGEN_AOAI_API_KEY=your-image-api-key
LLM_AOAI_API_KEY=your-llm-api-key
AZURE_STORAGE_ACCOUNT_KEY=your-storage-account-key
AZURE_COSMOS_DB_KEY=your-key
```

2. Line 36 has typo: `AZURE_COMOS_DB_ID` should be `AZURE_COSMOS_DB_ID`

3. Missing documentation for `AZURE_CLIENT_ID` (required for managed identity)

##### Best Practice Documents

✅ **Excellent Security Documentation:**
- `.github/azure-bestpractices.md` (361 lines) - Comprehensive zero-trust guidance
- `.github/bicep-deployment-bestpractices.md` (625 lines) - Detailed IaC best practices
- `.github/copilot-instructions.md` (110 lines) - Development standards

**Key Guidance:**
- **Zero API keys policy** clearly documented
- ChainedTokenCredential pattern explained
- Managed identity implementation examples
- RBAC configuration guidelines

#### Inconsistency Identified:

⚠️ **CRITICAL INCONSISTENCY:**
- Documentation promotes zero API keys policy
- Infrastructure implementation uses API keys
- Creates confusion for developers

#### Recommendations:

**HIGH PRIORITY:**

1. **Align .env.example with best practices:**

Remove API key variables and add managed identity configuration:
```bash
# Azure OpenAI Configuration (No API Keys!)
SORA_AOAI_RESOURCE=your-resource-name
SORA_DEPLOYMENT=your-sora-deployment-name

IMAGEGEN_AOAI_RESOURCE=your-image-resource-name
IMAGEGEN_DEPLOYMENT=your-image-deployment-name

LLM_AOAI_RESOURCE=your-llm-resource-name
LLM_DEPLOYMENT=your-llm-deployment-name

# Managed Identity Configuration
AZURE_CLIENT_ID=your-managed-identity-client-id  # Required for Azure Container Apps

# Azure Blob Storage (Endpoint only, no keys!)
AZURE_BLOB_SERVICE_URL=https://<storage-account>.blob.core.windows.net/
AZURE_STORAGE_ACCOUNT_NAME=your-storage-account-name
AZURE_BLOB_IMAGE_CONTAINER=images
AZURE_BLOB_VIDEO_CONTAINER=videos

# Azure Cosmos DB (Endpoint only, no keys!)
AZURE_COSMOS_DB_ENDPOINT=https://<cosmos-account>.documents.azure.com:443/
AZURE_COSMOS_DB_ID=visionarylab
AZURE_COSMOS_CONTAINER_ID=metadata
USE_MANAGED_IDENTITY=true
```

2. **Fix typo in .env.example:**
   - Line 36: Change `AZURE_COMOS_DB_ID` to `AZURE_COSMOS_DB_ID`

3. **Add security notice to README:**

```markdown
## 🔐 Security Note

This repository follows Azure zero-trust security practices:
- **No API keys or connection strings** in production deployments
- **Managed Identity** authentication for all Azure services
- All credentials retrieved securely via Azure Key Vault or RBAC

For local development, use Azure Developer CLI authentication (`azd auth login`).
```

4. **Add deployment validation section to DEPLOYMENT.md:**

```markdown
## Post-Deployment Validation

Verify your deployment:

```bash
# Check deployment status
azd show

# Verify container apps are running
az containerapp list --resource-group rg-<env-name> --output table

# Check logs for authentication issues
azd logs --follow

# Verify managed identity is configured
az identity list --resource-group rg-<env-name> --output table
```
```

5. **Document migration path from API keys to managed identity:**

Add a new section in DEPLOYMENT.md:

```markdown
## Migrating from API Keys to Managed Identity

If you have an existing deployment using API keys:

1. **Update infrastructure:**
   ```bash
   # Pull latest changes with managed identity support
   git pull origin main
   
   # Re-provision infrastructure
   azd provision
   ```

2. **Remove API key secrets:**
   ```bash
   azd env unset IMAGEGEN_AOAI_API_KEY
   azd env unset LLM_AOAI_API_KEY
   azd env unset SORA_AOAI_API_KEY
   ```

3. **Verify managed identity authentication:**
   ```bash
   # Check container app logs
   azd logs --service backend --follow
   ```
```

---

### 5. Environment Configuration

**Category:** configuration  
**Status:** ⚠️ PARTIALLY COMPLIANT  
**Description:** Environment variables are documented but include security anti-patterns.

#### Findings:

##### Environment Variable Sources:

1. **`.env.example` (45 lines):**
   - ✅ Comprehensive template for local development
   - ⚠️ Includes API key variables (security concern)
   - ⚠️ Typo: `AZURE_COMOS_DB_ID` (line 36)
   - ❌ Missing `AZURE_CLIENT_ID` for managed identity

2. **`infra/main.parameters.json` (51 lines):**
   - ✅ Proper azd variable substitution
   - ✅ Links to azd environment variables
   - ⚠️ Includes API key parameters

3. **`infra/modules/containerApp.bicep` (Lines 107-212):**
   - ✅ 27 environment variables configured
   - ⚠️ Includes API key environment variables
   - ✅ `USE_MANAGED_IDENTITY` set to `'true'` (line 206)
   - ❌ Missing `AZURE_CLIENT_ID` environment variable

##### Environment Variable Categories:

**Azure OpenAI Services (9 variables):**
- ✅ `IMAGEGEN_AOAI_RESOURCE`, `IMAGEGEN_DEPLOYMENT`
- ✅ `LLM_AOAI_RESOURCE`, `LLM_DEPLOYMENT`
- ✅ `SORA_AOAI_RESOURCE`, `SORA_DEPLOYMENT`
- ⚠️ `IMAGEGEN_AOAI_API_KEY` (should use managed identity)
- ⚠️ `LLM_AOAI_API_KEY` (should use managed identity)
- ⚠️ `SORA_AOAI_API_KEY` (should use managed identity)

**Azure Storage (4 variables):**
- ✅ `AZURE_BLOB_SERVICE_URL`
- ✅ `AZURE_STORAGE_ACCOUNT_NAME`
- ⚠️ `AZURE_STORAGE_ACCOUNT_KEY` (should use managed identity)
- ✅ `AZURE_BLOB_IMAGE_CONTAINER`

**Azure Cosmos DB (4 variables):**
- ✅ `AZURE_COSMOS_DB_ENDPOINT`
- ✅ `AZURE_COSMOS_DB_ID`
- ✅ `AZURE_COSMOS_CONTAINER_ID`
- ✅ `USE_MANAGED_IDENTITY` (correctly set to 'true')

**API Configuration (3 variables):**
- ✅ `API_PROTOCOL`, `API_HOSTNAME`, `API_PORT`

**Frontend Variables (4 variables):**
- ✅ `NEXT_PUBLIC_API_PROTOCOL`
- ✅ `NEXT_PUBLIC_API_HOSTNAME`
- ✅ `NEXT_PUBLIC_API_PORT`
- ✅ `STORAGE_ACCOUNT_NAME`

**Container Registry (1 variable):**
- ✅ `AZURE_CONTAINER_REGISTRY_ENDPOINT`

**Other (2 variables):**
- ✅ `MODEL_PROVIDER` (set to 'azure')

##### Secrets Management:

✅ **Good Practices:**
- Container Registry password uses secrets (line 79-84 in containerApp.bicep)
- Parameters use `@secure()` decorator

⚠️ **Security Concerns:**
- API keys stored in environment variables
- No Azure Key Vault integration for runtime secrets
- Secrets visible in container app configuration

#### Inconsistencies Found:

1. **Cosmos DB Configuration:**
   - `USE_MANAGED_IDENTITY` is set to `'true'` (correct)
   - But other services don't follow the same pattern
   - Cosmos DB has proper RBAC (good), but OpenAI and Storage don't

2. **Missing Environment Variables:**
   - `AZURE_CLIENT_ID` is documented as required but not set
   - `IMAGEGEN_15_DEPLOYMENT` and `IMAGEGEN_1_MINI_DEPLOYMENT` defined in main.bicep but not in .env.example

3. **Documentation Mismatch:**
   - README.md line 119 mentions Cosmos DB with `USE_MANAGED_IDENTITY`
   - But doesn't mention same pattern for OpenAI and Storage

#### Recommendations:

**CRITICAL:**

1. **Add AZURE_CLIENT_ID to all container apps:**

```bicep
// In containerApp.bicep, add at the beginning of env array:
{
  name: 'AZURE_CLIENT_ID'
  value: managedIdentityClientId  // Pass this as a parameter
}
```

2. **Remove all API key environment variables:**

Delete these from containerApp.bicep:
- `IMAGEGEN_AOAI_API_KEY`
- `LLM_AOAI_API_KEY`
- `SORA_AOAI_API_KEY`
- `AZURE_STORAGE_ACCOUNT_KEY`

3. **Update .env.example to match:**

```bash
# Azure Managed Identity (Required for Container Apps)
AZURE_CLIENT_ID=00000000-0000-0000-0000-000000000000

# Azure OpenAI - No API Keys!
SORA_AOAI_RESOURCE=your-resource-name
SORA_DEPLOYMENT=sora-2

IMAGEGEN_AOAI_RESOURCE=your-resource-name
IMAGEGEN_DEPLOYMENT=gpt-image-1
IMAGEGEN_15_DEPLOYMENT=gpt-image-1-5  # Optional
IMAGEGEN_1_MINI_DEPLOYMENT=gpt-image-1-mini  # Optional

LLM_AOAI_RESOURCE=your-resource-name
LLM_DEPLOYMENT=gpt-4.1

# Azure Storage - No Access Keys!
AZURE_BLOB_SERVICE_URL=https://yourstorage.blob.core.windows.net/
AZURE_STORAGE_ACCOUNT_NAME=yourstorage
AZURE_BLOB_IMAGE_CONTAINER=images
AZURE_BLOB_VIDEO_CONTAINER=videos

# Azure Cosmos DB - Managed Identity Enabled
AZURE_COSMOS_DB_ENDPOINT=https://yourcosmosdb.documents.azure.com:443/
AZURE_COSMOS_DB_ID=visionarylab  # Fixed typo from COMOS
AZURE_COSMOS_CONTAINER_ID=metadata
USE_MANAGED_IDENTITY=true
```

4. **Document environment variables in a dedicated file:**

Create `docs/ENVIRONMENT_VARIABLES.md`:

```markdown
# Environment Variables Reference

## Azure Identity

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| AZURE_CLIENT_ID | Yes (Container Apps) | Managed Identity Client ID | 00000000-0000-0000-0000-000000000000 |

## Azure OpenAI

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| LLM_AOAI_RESOURCE | Yes | OpenAI resource name | my-openai-resource |
| LLM_DEPLOYMENT | Yes | LLM model deployment | gpt-4.1 |
| IMAGEGEN_AOAI_RESOURCE | Yes | OpenAI resource for images | my-openai-resource |
| IMAGEGEN_DEPLOYMENT | Yes | Image model deployment | gpt-image-1 |
| SORA_AOAI_RESOURCE | Yes | OpenAI resource for Sora | my-openai-resource |
| SORA_DEPLOYMENT | Yes | Sora model deployment | sora-2 |

**Note:** API keys are NOT required. Authentication uses Managed Identity with RBAC.

[Continue with other services...]
```

---

## Compliance Summary

### By Category

| Category | Status | Score | Critical Issues |
|----------|--------|-------|-----------------|
| azure.yaml | ✅ Compliant | 9/10 | Minor path inconsistency |
| Infrastructure | ⚠️ Partially Compliant | 6/10 | API keys, missing RBAC |
| Workflows | ✅ Compliant | 10/10 | None |
| Documentation | ✅ Compliant | 8/10 | Inconsistency with implementation |
| Configuration | ⚠️ Partially Compliant | 6/10 | API keys, missing CLIENT_ID |

### Priority Recommendations

#### 🔴 **CRITICAL (Must Fix Before Production)**

1. **Remove all API keys from infrastructure templates**
   - Impact: Security vulnerability, non-compliance with Azure best practices
   - Effort: Medium (requires infrastructure refactoring)
   - Files: `infra/main.bicep`, `infra/modules/containerApp.bicep`, `infra/main.parameters.json`

2. **Implement User Assigned Managed Identity**
   - Impact: Enables secure authentication to Azure services
   - Effort: Medium (requires new module and RBAC configuration)
   - Files: New `infra/modules/userAssignedIdentity.bicep`, update `infra/main.bicep`

3. **Add RBAC role assignments for Azure OpenAI and Storage**
   - Impact: Required for managed identity authentication
   - Effort: Low (add resource blocks)
   - Files: `infra/main.bicep`

4. **Add AZURE_CLIENT_ID to container app environment variables**
   - Impact: Required for ChainedTokenCredential to work
   - Effort: Low (add one environment variable)
   - Files: `infra/modules/containerApp.bicep`

#### 🟡 **HIGH PRIORITY (Recommended)**

5. **Update .env.example to remove API keys**
   - Impact: Prevents developers from thinking API keys are required
   - Effort: Low
   - Files: `.env.example`

6. **Fix typo AZURE_COMOS_DB_ID → AZURE_COSMOS_DB_ID**
   - Impact: Prevents configuration errors
   - Effort: Very Low
   - Files: `.env.example`

7. **Add missing environment variables to .env.example**
   - Variables: `AZURE_CLIENT_ID`, `IMAGEGEN_15_DEPLOYMENT`, `IMAGEGEN_1_MINI_DEPLOYMENT`
   - Impact: Complete documentation
   - Effort: Very Low
   - Files: `.env.example`

8. **Update frontend Dockerfile path in azure.yaml**
   - Impact: Consistency and clarity
   - Effort: Very Low
   - Files: `azure.yaml`

#### 🟢 **MEDIUM PRIORITY (Nice to Have)**

9. **Add security notice to README**
   - Impact: Clear communication of security model
   - Effort: Low
   - Files: `README.md`

10. **Create environment variables reference document**
    - Impact: Improved developer experience
    - Effort: Medium
    - Files: New `docs/ENVIRONMENT_VARIABLES.md`

11. **Enable automatic deployment workflow triggers**
    - Impact: CI/CD automation
    - Effort: Low
    - Files: `.github/workflows/azure-dev.yaml`

12. **Add deployment validation documentation**
    - Impact: Better troubleshooting
    - Effort: Low
    - Files: `DEPLOYMENT.md`

---

## Compliance Checklist

### ✅ Compliant Items (9/14)

- [x] azure.yaml file present with valid schema
- [x] Infrastructure templates exist in /infra directory
- [x] Infrastructure templates use Bicep (recommended provider)
- [x] Deployment workflows reference azd commands correctly
- [x] Service definitions match application structure
- [x] README documents azd up/deploy commands
- [x] DEPLOYMENT.md provides detailed instructions
- [x] Post-provision hooks are defined
- [x] GitHub Actions uses federated credentials

### ⚠️ Non-Compliant Items (5/14)

- [ ] **Infrastructure uses API keys (violates best practices)**
- [ ] **Missing User Assigned Managed Identity**
- [ ] **Missing RBAC role assignments for OpenAI**
- [ ] **Missing RBAC role assignments for Storage**
- [ ] **Container Apps missing AZURE_CLIENT_ID environment variable**

---

## Testing Recommendations

To validate azd compliance, run these tests:

### 1. Infrastructure Validation

```bash
# Validate Bicep syntax
az bicep build --file infra/main.bicep

# Lint Bicep files
az bicep lint --file infra/main.bicep

# What-if deployment (preview changes)
az deployment group what-if \
  --resource-group rg-test \
  --template-file infra/main.bicep \
  --parameters infra/main.parameters.json
```

### 2. azd Command Testing

```bash
# Initialize new environment
azd env new test-compliance

# Validate azd configuration
azd config list-alpha

# Test provision (dry run if possible)
azd provision --preview  # If supported

# Verify environment variables
azd env get-values

# Test deployment
azd deploy --service backend
azd deploy --service frontend
```

### 3. Post-Deployment Validation

```bash
# Verify managed identity is assigned
az containerapp show \
  --name ca-backend-test \
  --resource-group rg-test \
  --query identity

# Verify RBAC assignments
az role assignment list \
  --assignee <managed-identity-principal-id> \
  --all

# Check container app logs for auth errors
azd logs --service backend --follow

# Verify container apps are running
az containerapp list \
  --resource-group rg-test \
  --output table
```

### 4. Security Validation

```bash
# Scan for exposed secrets in git history
git secrets --scan-history  # Requires git-secrets

# Check for API keys in environment variables
az containerapp show \
  --name ca-backend-test \
  --resource-group rg-test \
  --query 'properties.template.containers[0].env[*].name' \
  | grep -i "key\|secret"

# Should return empty or only allowed keys like registry secrets
```

---

## Implementation Roadmap

### Phase 1: Security Remediation (Week 1)
**Goal:** Remove all API keys and implement managed identity

- Day 1-2: Create userAssignedIdentity module
- Day 3-4: Update main.bicep with RBAC assignments
- Day 5: Remove API key parameters and environment variables
- Day 6: Update containerApp.bicep with AZURE_CLIENT_ID
- Day 7: Testing and validation

### Phase 2: Documentation Updates (Week 2)
**Goal:** Align documentation with implementation

- Day 1: Update .env.example
- Day 2: Fix typos and add missing variables
- Day 3: Update README with security notice
- Day 4: Enhance DEPLOYMENT.md
- Day 5: Create ENVIRONMENT_VARIABLES.md reference
- Day 6-7: Review and final validation

### Phase 3: CI/CD Enhancements (Week 3)
**Goal:** Improve automation and monitoring

- Day 1-2: Enable automatic workflow triggers
- Day 3-4: Add deployment validation checks
- Day 5: Implement workflow status monitoring
- Day 6-7: Documentation and training

---

## Application Code Considerations

**Note:** This compliance review focused on azd configuration. The following application-level changes are also required for managed identity:

### Backend (Python)

Update authentication to use ChainedTokenCredential:

```python
# backend/core/auth.py
from azure.identity import AzureDeveloperCliCredential, ManagedIdentityCredential, ChainedTokenCredential
from azure.ai.openai import AzureOpenAI
from azure.storage.blob import BlobServiceClient
from azure.cosmos import CosmosClient

def get_azure_credential():
    """Get Azure credential using best practice chain."""
    return ChainedTokenCredential(
        AzureDeveloperCliCredential(),  # Local development
        ManagedIdentityCredential()     # Production
    )

# Use with Azure services
credential = get_azure_credential()

# Azure OpenAI
openai_client = AzureOpenAI(
    azure_endpoint=os.getenv("LLM_AOAI_RESOURCE"),
    azure_ad_token_provider=get_bearer_token_provider(
        credential, 
        "https://cognitiveservices.azure.com/.default"
    )
)

# Azure Storage
blob_client = BlobServiceClient(
    account_url=os.getenv("AZURE_BLOB_SERVICE_URL"),
    credential=credential
)

# Azure Cosmos DB
cosmos_client = CosmosClient(
    url=os.getenv("AZURE_COSMOS_DB_ENDPOINT"),
    credential=credential
)
```

### Frontend (Node.js/Next.js)

Frontend typically doesn't need direct Azure authentication since it calls the backend API. However, if frontend needs to access Azure Storage directly:

```typescript
// frontend/utils/azure-auth.ts
import { 
  ChainedTokenCredential, 
  AzureDeveloperCliCredential, 
  ManagedIdentityCredential 
} from '@azure/identity';

export function getAzureCredential(): ChainedTokenCredential {
  return new ChainedTokenCredential(
    new AzureDeveloperCliCredential(),
    new ManagedIdentityCredential()
  );
}
```

---

## Additional Resources

### Microsoft Documentation
- [Azure Developer CLI documentation](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- [Azure Container Apps documentation](https://learn.microsoft.com/azure/container-apps/)
- [Managed identities for Azure resources](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
- [Azure RBAC built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)

### Repository Resources
- [Azure Best Practices](.github/azure-bestpractices.md)
- [Bicep Deployment Best Practices](.github/bicep-deployment-bestpractices.md)
- [GitHub Copilot Instructions](.github/copilot-instructions.md)

---

## Conclusion

The Visionary Lab repository demonstrates **strong foundational compliance** with Azure Developer CLI requirements. The project structure, infrastructure templates, and deployment workflows are well-implemented and follow azd conventions.

However, **critical security improvements** are required before production deployment:

1. **Eliminate API key usage** throughout the infrastructure
2. **Implement managed identity** for all Azure service authentication
3. **Configure proper RBAC** role assignments
4. **Update documentation** to align with implementation

Once these security concerns are addressed, this repository will serve as an excellent example of azd-compliant infrastructure and deployment practices.

**Estimated effort to achieve full compliance:** 2-3 weeks for one developer

**Current compliance level:** 75% (Partial Compliance)  
**Target compliance level:** 100% (Full Compliance)

---

**Report End**
