# DevOps Infrastructure Design — Azure + Cloudflare

## Overview

This document describes a production-grade infrastructure design on **Microsoft Azure** with **Cloudflare at the edge**, supporting:

- Web app (central configuration UI)
- Backend APIs
- Log ingestion & processing from in-venue devices
- Video processing and serving for in-game moments
- SMS & Email sending to guests
- Asynchronous device ↔ backend communication
- CI/CD pipelines and Infrastructure as Code

---

## Architecture Diagram (Textual)

```
[In-Venue Devices]
       │
       ├──── MQTT / AMQP ──────────────────► Azure Service Bus / Event Hub
       │                                            │
       └──── HTTPS logs ──►  Cloudflare WAF ──►  Azure API Management
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                   [Config Web App]      [Backend APIs]
                  (Azure Static Web    (Azure Container Apps /
                   Apps / App Svc)       AKS — containerised)
                         │                     │
                    Azure AD B2C         Azure Service Bus
                                               │
                         ┌─────────────────────┤
                         │                     │
                   [Log Processor]       [Video Processor]
                (Azure Functions /     (Azure Media Services /
                 Stream Analytics)       Container Jobs)
                         │                     │
                   Azure Monitor         Azure Blob Storage
                   Log Analytics         + Azure CDN / CF
                         │
                   [Notification Workers]
                   SMS: Azure Communication Services / Twilio
                   Email: Azure Communication Services / SendGrid
```

---

## Assumptions & Trade-offs

| Assumption | Trade-off |
|---|---|
| In-venue devices are constrained IoT/edge hardware (MQTT preferred) | MQTT over Azure Service Bus is simpler than raw AMQP; slightly less throughput headroom |
| Video moments are short clips (highlights), not live streams | Azure Media Services / Blob + CDN is cost-effective; full live streaming would need Azure Live Events (expensive) |
| Guest volume per venue peaks at ~50 k concurrent | Container Apps auto-scaling handles this; AKS gives more control at the cost of more ops overhead |
| Multi-region is a day-2 concern; primary region: UK South | Single-region simplifies day-1; geo-replication of Cosmos DB / Service Bus added later |
| Internal staff use the config web app (not public) | Azure AD (Entra ID) used; no need for B2C on the config app |
| SMS/Email volumes are burst, not continuous | Managed services (Azure Communication Services) avoid maintaining own SMTP/SMPP |
| IaC tool: Terraform (open source, provider-agnostic) | Bicep is Azure-native and simpler, but Terraform skills are more portable |

---

## Service Map

### 1. Edge — Cloudflare

- **WAF**: Block OWASP Top 10, rate-limit per IP/venue
- **DDoS**: Cloudflare Magic Transit / HTTP DDoS protection
- **CDN**: Cache static assets and video segments at PoP closest to venue
- **DNS**: Cloudflare DNS with orange-cloud proxying for all public endpoints
- **Workers** (optional): JWT pre-validation at edge, A/B routing, geo-based redirects

### 2. Config Web App

- **Hosting**: Azure Static Web Apps (if React/Vue SPA) or Azure App Service (if SSR)
- **Auth**: Microsoft Entra ID (Azure AD) — OIDC/OAuth2, staff login only
- **Backend**: Calls Backend APIs via APIM

### 3. Backend APIs

- **Hosting**: Azure Container Apps (preferred) — serverless containers with built-in KEDA scaling
- **Gateway**: Azure API Management (APIM) — auth, throttling, versioning, developer portal
- **Language**: Any (Node.js / Python / .NET — containerised)
- **Data**: Azure Cosmos DB (NoSQL, globally distributed) or Azure PostgreSQL Flexible Server

### 4. Log Ingestion & Processing

- **Ingest**: Devices POST logs to APIM endpoint → Azure Event Hubs (high-throughput, partitioned)
- **Processing**: Azure Stream Analytics (SQL-based, real-time) or Azure Functions (event-driven)
- **Storage**: Azure Data Lake Storage Gen2 (raw) + Azure Synapse / Fabric (analytics)
- **Alerting**: Log Analytics Workspace → Azure Monitor Alerts

### 5. Video Processing & Serving

- **Upload**: Devices or backend push raw clips to Azure Blob Storage (private container) via SAS tokens
- **Processing**: Azure Container Apps Jobs (FFmpeg) triggered by Blob events via Event Grid
  - Transcode to HLS/DASH adaptive bitrate
  - Generate thumbnails
  - Store outputs to public Blob container
- **Serving**: Azure CDN (or Cloudflare in front of Blob) — cache video segments at edge

### 6. SMS Sending

- **Service**: Azure Communication Services (SMS) — managed, no infra, pay-per-message
- **Fallback**: Twilio (via abstraction layer in code so provider is swappable)
- **Queue**: Notifications requested via Service Bus queue → worker Container App sends SMS

### 7. Email Sending

- **Service**: Azure Communication Services (Email) with custom domain + DKIM/SPF/DMARC
- **Fallback**: SendGrid (Azure Marketplace integration)
- **Queue**: Same pattern — Service Bus → worker → send

### 8. Async Device ↔ Backend Communication

- **Protocol**: MQTT via Azure IoT Hub (managed, scales to millions of devices, built-in device registry) or Azure Service Bus (simpler, if devices support AMQP/HTTPS)
- **Pattern**: Devices publish to topic → IoT Hub routes to Service Bus → backend workers consume
- **Commands back to devices**: IoT Hub direct methods or C2D messages

---

## Infrastructure as Code

All infrastructure defined in **Terraform**, structured as:

```
infra/
├── main.tf                  # Root module, calls child modules
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
├── backend.tf               # Azure Storage remote state
└── modules/
    ├── networking/          # VNet, subnets, NSGs, Private Endpoints
    ├── cloudflare/          # DNS records, WAF rules, Page Rules
    ├── apim/                # API Management instance + APIs
    ├── container_apps/      # Container Apps Environment + apps
    ├── event_hubs/          # Event Hub Namespace + hubs
    ├── service_bus/         # Service Bus Namespace + queues/topics
    ├── iot_hub/             # IoT Hub + routing rules
    ├── storage/             # Blob containers, lifecycle policies
    ├── cdn/                 # Azure CDN profile + endpoints
    ├── cosmos_db/           # Cosmos DB account + databases
    ├── monitoring/          # Log Analytics, App Insights, Alerts
    ├── communication/       # Azure Communication Services
    └── identity/            # Managed Identities, Key Vault, RBAC
```

### Key Terraform Snippets

```hcl
# modules/container_apps/main.tf
resource "azurerm_container_app_environment" "main" {
  name                       = "cae-${var.env}-${var.location_short}"
  location                   = var.location
  resource_group_name        = var.resource_group_name
  log_analytics_workspace_id = var.log_analytics_id
}

resource "azurerm_container_app" "api" {
  name                         = "ca-api-${var.env}"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = var.resource_group_name
  revision_mode                = "Multiple"

  ingress {
    external_enabled = true
    target_port      = 8080
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }

  template {
    min_replicas = 1
    max_replicas = 20

    http_scale_rule {
      name                = "http-scaler"
      concurrent_requests = "100"
    }

    container {
      name   = "api"
      image  = "${var.acr_login_server}/api:${var.image_tag}"
      cpu    = 0.5
      memory = "1Gi"

      env {
        name        = "SERVICE_BUS_CONNECTION"
        secret_name = "sb-connection"
      }
    }
  }

  secret {
    name  = "sb-connection"
    value = var.service_bus_connection_string
  }
}
```

```hcl
# modules/event_hubs/main.tf
resource "azurerm_eventhub_namespace" "logs" {
  name                = "evhns-logs-${var.env}"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "Standard"
  capacity            = 2  # throughput units; auto-inflate enabled
  auto_inflate_enabled     = true
  maximum_throughput_units = 10
}

resource "azurerm_eventhub" "device_logs" {
  name                = "device-logs"
  namespace_name      = azurerm_eventhub_namespace.logs.name
  resource_group_name = var.resource_group_name
  partition_count     = 32
  message_retention   = 7
}
```

---

## CI/CD Pipelines

All pipelines live in **Azure DevOps (ADO)**, using YAML pipelines stored in the repo alongside the code. ADO integrates natively with Azure (service connections, ACR, Key Vault variable groups) and is the natural choice given the Azure-first stack.

### ADO Setup

- **Organisation**: one ADO org, one project per product
- **Service connections**: Azure Resource Manager (Workload Identity Federation — no client secrets) per environment; Docker Registry connection to ACR
- **Variable groups**: linked to Azure Key Vault per environment (`vg-dev`, `vg-staging`, `vg-production`) — secrets pulled at runtime, never stored in ADO
- **Environments**: ADO Environments (`staging`, `production`) with approval gates on production
- **Agent pool**: Microsoft-hosted `ubuntu-latest` for most jobs; self-hosted agents in the VNet for jobs that need private endpoint access

### Pipeline Structure

```
pipelines/
├── ci.yml              # On PR: lint, test, build, security scan, push to ACR
├── cd-staging.yml      # On merge to main: deploy to staging + smoke tests
├── cd-production.yml   # On release branch/tag: deploy to production (with approval)
└── infra.yml           # On infra/ changes: terraform plan (PR) / apply (merge)
```

### CI Pipeline — `pipelines/ci.yml`

Triggered on every pull request targeting `main`.

```yaml
trigger: none
pr:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-staging          # Key Vault-linked variable group (ACR_LOGIN_SERVER, ACR_NAME)
  - name: imageTag
    value: $(Build.SourceVersion)  # Git SHA

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - checkout: self

          - script: make test
            displayName: Run unit tests

          - task: Docker@2
            displayName: Build Docker image
            inputs:
              command: build
              repository: $(ACR_LOGIN_SERVER)/api
              tags: $(imageTag)
              Dockerfile: apps/api/Dockerfile

          - script: |
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
              trivy image --severity CRITICAL,HIGH --exit-code 1 $(ACR_LOGIN_SERVER)/api:$(imageTag)
            displayName: Trivy security scan

          - task: Docker@2
            displayName: Push image to ACR
            inputs:
              command: push
              containerRegistry: sc-acr-staging   # ADO service connection
              repository: api
              tags: $(imageTag)
```

### CD Staging Pipeline — `pipelines/cd-staging.yml`

Triggered automatically on merge to `main`.

```yaml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-staging
  - name: imageTag
    value: $(Build.SourceVersion)

stages:
  - stage: DeployStaging
    displayName: Deploy to Staging
    jobs:
      - deployment: DeployAPI
        environment: staging          # ADO Environment — records deployment history
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: Update Container App revision
                  inputs:
                    azureSubscription: sc-azure-staging   # ARM service connection (WIF)
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp update \
                        --name ca-api-staging \
                        --resource-group rg-app-staging \
                        --image $(ACR_LOGIN_SERVER)/api:$(imageTag)

                - script: make smoke-test ENVIRONMENT=staging
                  displayName: Smoke tests

  - stage: NotifyOnFailure
    condition: failed()
    jobs:
      - job: Notify
        steps:
          - task: InvokeRestAPI@1
            displayName: Post to Teams webhook
            inputs:
              connectionType: connectedServiceName
              serviceConnection: sc-teams-webhook
              method: POST
              body: '{"text":"Staging deploy failed — $(Build.BuildNumber)"}'
```

### CD Production Pipeline — `pipelines/cd-production.yml`

Triggered on a release branch (`release/*`) or manual tag. Requires an ADO approval gate before deploying.

```yaml
trigger:
  branches:
    include: [release/*]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-production
  - name: imageTag
    value: $(Build.SourceVersion)

stages:
  - stage: DeployProduction
    displayName: Deploy to Production
    jobs:
      - deployment: DeployAPI
        environment: production       # Has approval check configured in ADO UI
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: Blue/green — route 10% traffic to new revision
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp revision copy \
                        --name ca-api-production \
                        --resource-group rg-app-production \
                        --image $(ACR_LOGIN_SERVER)/api:$(imageTag)

                      NEW_REVISION=$(az containerapp revision list \
                        --name ca-api-production \
                        --resource-group rg-app-production \
                        --query "[0].name" -o tsv)

                      az containerapp ingress traffic set \
                        --name ca-api-production \
                        --resource-group rg-app-production \
                        --revision-weight latest=10 previous=90

                - script: make smoke-test ENVIRONMENT=production
                  displayName: Production smoke tests

                - task: AzureCLI@2
                  displayName: Shift 100% traffic to new revision
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp ingress traffic set \
                        --name ca-api-production \
                        --resource-group rg-app-production \
                        --revision-weight latest=100
```

### Infrastructure Pipeline — `pipelines/infra.yml`

PR → `terraform plan` output posted as PR comment. Merge to `main` → `terraform apply`.

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: [infra/**]

pr:
  paths:
    include: [infra/**]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-staging          # contains ARM_CLIENT_ID, ARM_TENANT_ID etc. (WIF)
  - name: TF_WORKING_DIR
    value: infra/

stages:
  - stage: TerraformPlan
    condition: eq(variables['Build.Reason'], 'PullRequest')
    jobs:
      - job: Plan
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: latest

          - task: TerraformTaskV4@4
            displayName: Terraform Init
            inputs:
              provider: azurerm
              command: init
              workingDirectory: $(TF_WORKING_DIR)
              backendServiceArm: sc-azure-staging
              backendAzureRmResourceGroupName: rg-tfstate
              backendAzureRmStorageAccountName: stgtfstate
              backendAzureRmContainerName: tfstate
              backendAzureRmKey: staging.terraform.tfstate

          - task: TerraformTaskV4@4
            displayName: Terraform Plan
            inputs:
              provider: azurerm
              command: plan
              workingDirectory: $(TF_WORKING_DIR)
              environmentServiceNameAzureRM: sc-azure-staging
              commandOptions: -out=tfplan

          - script: |
              terraform show -no-color tfplan > plan.txt
              echo "##vso[task.uploadsummary]$(System.DefaultWorkingDirectory)/plan.txt"
            displayName: Post plan to PR summary
            workingDirectory: $(TF_WORKING_DIR)

  - stage: TerraformApply
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - deployment: Apply
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  displayName: Terraform Apply
                  inputs:
                    provider: azurerm
                    command: apply
                    workingDirectory: $(TF_WORKING_DIR)
                    environmentServiceNameAzureRM: sc-azure-staging
                    commandOptions: -auto-approve
```

---

## Security

| Layer | Control |
|---|---|
| **Edge** | Cloudflare WAF (OWASP ruleset), DDoS, bot management, mTLS for device-to-cloud |
| **Network** | Azure VNet with private subnets; all PaaS services on Private Endpoints (no public IPs) |
| **Identity** | Managed Identities for all Azure services (no credentials in code); Entra ID for staff |
| **Secrets** | Azure Key Vault; secrets injected at runtime via environment variables or Key Vault references |
| **API** | APIM enforces JWT validation, per-subscription rate limits, IP allow-listing for venue devices |
| **Containers** | Non-root containers, read-only filesystems, Trivy scans in CI |
| **Data** | Encryption at rest (Azure managed keys, or CMK for sensitive data); TLS 1.2+ in transit |
| **IoT Devices** | Per-device X.509 certs in IoT Hub; certificate rotation via DPS |
| **Compliance** | Azure Policy enforces tagging, allowed regions, required diagnostics settings |

---

## Scalability

- **Container Apps**: KEDA-based autoscaling on HTTP requests, Service Bus queue depth, or Event Hub lag
- **Event Hubs**: Partition count set at creation (32); throughput units auto-inflate
- **Service Bus**: Premium tier supports 1–16 messaging units, scale independently per queue
- **Cosmos DB**: Autoscale RU/s (400–40,000 by default); partition key chosen for even distribution
- **Video**: Container App Jobs scale out FFmpeg workers independently; Blob Storage is unlimited
- **CDN**: Azure CDN / Cloudflare caches video segments and static assets at edge — origin traffic drops ~80–90%

---

## Reliability

- **Target SLA**: 99.9% (composed SLA across Azure Container Apps + APIM + Cosmos DB multi-region reads)
- **Health checks**: APIM health probes + Container Apps liveness/readiness probes
- **Retry policies**: Exponential backoff on Service Bus consumers; dead-letter queues for poison messages
- **Circuit breaker**: APIM policies + client-side (Polly in .NET, retry libraries in other stacks)
- **Chaos testing**: Azure Chaos Studio for failure injection in staging

---

## Resiliency

- **Redundancy**: Container Apps run across Azure Availability Zones (zone-redundant environment)
- **Data**: Cosmos DB multi-region reads (write to UK South, read replicas in West Europe); geo-redundant Blob
- **Service Bus / Event Hubs**: Geo-disaster recovery pairing (secondary namespace in West Europe)
- **IoT Hub**: Manual failover to secondary region (built-in)
- **Video**: CDN serves cached segments even if origin is down; long TTLs on immutable video files
- **RPO/RTO**: RPO < 1 min (Cosmos continuous backup), RTO < 15 min (Container Apps redeploy)

---

## Observability

### Three Pillars

| Pillar | Tool |
|---|---|
| **Logs** | All services ship to central Log Analytics Workspace; structured JSON logs |
| **Metrics** | Azure Monitor metrics + custom app metrics via App Insights |
| **Traces** | Azure Application Insights distributed tracing (W3C TraceContext) |

### Dashboards & Alerts

- Azure Workbooks for operational dashboards (API latency p50/p95/p99, error rates, queue depth)
- Grafana (optional, via Azure Managed Grafana) for engineering dashboards
- PagerDuty / Opsgenie integration via Azure Monitor Action Groups
- Cloudflare Analytics for edge traffic visibility

### Key Alerts

- API error rate > 1% for 5 minutes → PagerDuty P2
- Event Hub consumer lag > 10,000 messages → Scale-out + alert
- Video processing job failure rate > 5% → Alert
- DLQ depth > 0 → Investigate immediately

---

## Automation

- **IaC**: All infrastructure in Terraform; no manual portal changes (enforced by Azure Policy `deny` on manual resource creation outside IaC)
- **GitOps**: Infrastructure changes via PR → plan posted as PR summary → review → merge → auto-apply
- **Config management**: Environment-specific values in Terraform workspaces + ADO Environments with approval gates; secrets in Azure Key Vault linked as ADO Variable Groups
- **Container builds**: Automated in CI on every commit; image tag = git SHA (`Build.SourceVersion`)
- **Dependency updates**: Renovate Bot (ADO-compatible) for container base images, Terraform providers, npm/pip packages
- **Certificate management**: Let's Encrypt via Cloudflare ACME (automatic renewal); IoT device certs rotated via Azure DPS enrollment groups
- **Cost management**: Azure Cost Management budgets + alerts; dev/staging resources auto-shut-down outside business hours via Azure Automation runbooks

---

## Environments

| Environment | Purpose | Scale | Notes |
|---|---|---|---|
| `dev` | Individual developer testing | Minimal (1 replica, shared) | Feature flags off |
| `staging` | Pre-production, smoke tests | ~20% of prod | Mirrors prod config |
| `production` | Live | Full autoscale | Blue/green via APIM traffic weights |

---

## Repo Structure

```
.
├── infra/                  # Terraform (see above)
├── apps/
│   ├── api/                # Backend API service
│   ├── config-web/         # Config web application
│   ├── log-processor/      # Log ingestion worker
│   ├── video-processor/    # Video transcoding worker
│   └── notification-worker/# SMS + Email sender
├── pipelines/
│   ├── ci.yml              # Build, test, scan, push to ACR
│   ├── cd-staging.yml      # Deploy to staging on merge to main
│   ├── cd-production.yml   # Deploy to production on release branch
│   └── infra.yml           # Terraform plan (PR) / apply (merge)
├── docs/
│   └── adr/                # Architecture Decision Records
└── README.md               # This file
```

---

## Key Architecture Decisions (ADRs)

1. **Container Apps over AKS**: Lower ops burden; KEDA scaling built-in; revisit if Kubernetes-specific features are needed
2. **Event Hubs for log ingestion over Kafka**: Native Azure integration; cheaper at moderate scale; Kafka-compatible protocol means future migration is straightforward
3. **Azure IoT Hub for device communication**: Built-in device registry, per-device auth, and C2D messaging justify the cost over raw Service Bus at scale
4. **Cloudflare for CDN over Azure CDN alone**: Better global PoP coverage, superior WAF, Workers for edge logic; Azure CDN remains as origin shield
5. **Azure Communication Services over Twilio**: Single vendor, lower latency to Azure backend, Azure-native billing; Twilio abstraction layer kept as fallback
