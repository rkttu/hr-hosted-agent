# Part 2: 엔터프라이즈 설정 — Hosted Agent + Foundry IQ + MAF 아키텍처

**[🇺🇸 English](README.md)** | **🇰🇷 한국어**

**같은 에이전트. 같은 코드. 엔터프라이즈급 인프라.**

이것은 [Hosted Agent 튜토리얼](../README.ko.md)의 **Part 2**입니다. Part 1과 동일한 HR 에이전트를 배포하되, 엔터프라이즈 인프라 위에서 실행합니다:

| 엔터프라이즈 기능 | 설명 |
| --- | --- |
| **고객 관리 키 (CMK)** | 모든 미사용 데이터가 YOUR Key Vault의 YOUR 키로 암호화 |
| **키리스 인증 (Managed Identity)** | API 키 제로 — 모든 인증이 User-Assigned Managed Identity + RBAC |
| **프라이빗 엔드포인트** | 모든 서비스가 VNET 뒤에 — 퍼블릭 인터넷 노출 없음 |

> **핵심 인사이트:** 애플리케이션 코드는 변경되지 않습니다. `main.py`, `deploy.py`, `Dockerfile` — 모두 Part 1과 동일합니다. 엔터프라이즈 보안은 전적으로 인프라 레이어 (Bicep)에 있습니다.

---

## 아키텍처

```text
┌──────────────────── Virtual Network (10.0.0.0/16) ────────────────────┐
│                                                                        │
│  Private Endpoints Subnet (10.0.1.0/24)                               │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                                                                │   │
│  │   ┌───────────────── Azure AI Foundry ─────────────────┐      │   │
│  │   │                                                     │      │   │
│  │   │  ┌──────────────┐       ┌──────────────────────┐   │      │   │
│  │   │  │ Hosted Agent  │──MI──▶│ AI Services (OpenAI)  │   │      │   │
│  │   │  │ (컨테이너)    │       │ 🔒 CMK │ 🔑 MI 전용   │   │      │   │
│  │   │  │  :8088        │       └──────────────────────┘   │      │   │
│  │   │  └──────┬───────┘                                   │      │   │
│  │   │         │              ┌──────────────────────┐     │      │   │
│  │   │         └─────MI──────▶│ AI Search            │     │      │   │
│  │   │                        │ 🔒 CMK │ 🔑 MI 전용   │     │      │   │
│  │   │                        └──────────────────────┘     │      │   │
│  │   └─────────────────────────────────────────────────────┘      │   │
│  │                                                                │   │
│  │   ┌──────────┐  ┌───────────┐  ┌──────────────────────┐      │   │
│  │   │ Key Vault │  │ Storage   │  │ Container Registry   │      │   │
│  │   │ 🔑 CMK 원본│  │ 🔒 CMK    │  │ 🔒 CMK │ Premium     │      │   │
│  │   │ 🔐 RBAC   │  │ 🔑 MI     │  │ 🔑 MI 전용           │      │   │
│  │   └──────────┘  └───────────┘  └──────────────────────┘      │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  🔐 User-Assigned Managed Identity (단일 ID, 모든 RBAC 역할)          │
│  🔑 Key Vault의 Customer-Managed Key (RSA-2048, 모든 미사용 데이터)   │
│  🔒 Private Endpoints + Private DNS Zones (퍼블릭 노출 제로)          │
└────────────────────────────────────────────────────────────────────────┘
```

### Part 1과의 차이점

| 항목 | Part 1 (표준) | Part 2 (엔터프라이즈) |
| --- | --- | --- |
| **네트워크** | 퍼블릭 엔드포인트 | 프라이빗 엔드포인트 + VNET |
| **암호화** | Microsoft 관리 키 | Key Vault의 고객 관리 키 (CMK) |
| **인증** | DefaultAzureCredential (모든 방법) | Managed Identity만 — 모든 서비스에서 API 키 비활성화 |
| **Key Vault** | 불필요 | RBAC 전용, 제거 보호 활성화, 소프트 삭제 활성화 |
| **ACR** | Basic SKU | Premium SKU (PE + CMK에 필요) |
| **AI Services** | `disableLocalAuth: false` | `disableLocalAuth: true` |
| **AI Search** | API 키 사용 가능 | `disableLocalAuth: true`, CMK 적용 활성화 |
| **Storage** | 공유 키 접근 | `allowSharedKeyAccess: false` |
| **서비스 인증** | 키 기반 연결 | ID 기반 (MI + RBAC, 키 없음) |
| **앱 코드** | `main.py` | **동일** — 변경 불필요 |

---

## 프로젝트 구조

```text
enterprise/
├── infra/                              # 🏗️  Bicep 인프라 코드
│   ├── main.bicep                      #     오케스트레이터 — 전체 배포
│   └── modules/
│       ├── managed-identity.bicep      #     User-assigned MI (단일 ID)
│       ├── network.bicep               #     VNET + 서브넷 + 7개 프라이빗 DNS 존
│       ├── keyvault.bicep              #     Key Vault + CMK 키 + PE
│       ├── storage.bicep               #     Storage + CMK + PE (blob + file)
│       ├── ai-services.bicep           #     AI Services + 모델 + Foundry 프로젝트 + CMK + PE
│       ├── ai-search.bicep             #     AI Search + CMK 적용 + PE
│       └── acr.bicep                   #     Container Registry + CMK + PE
├── main.py                             # ⭐ 에이전트 코드 (Part 1과 동일)
├── deploy.py                           # 🚀 Foundry에 에이전트 등록 (Part 1과 동일)
├── deploy-infra.ps1                    # 🏗️  모든 인프라를 배포하는 PowerShell 스크립트
├── Dockerfile                          # 🐳 컨테이너 이미지 (Part 1과 동일)
├── pyproject.toml                      # 📦 의존성 (uv 프로젝트 파일)
├── agent.yaml                          # 📄 에이전트 메타데이터
└── README.md                           # 📖 이 파일
```

---

## 사전 요구 사항

### Azure 권한

배포자에게 필요한 권한:

- 구독/리소스 그룹에 대한 **Contributor** (리소스 생성용)
- **User Access Administrator** (RBAC 역할 할당 생성용)
- 배포 스크립트가 CMK 키 생성을 위해 **Key Vault Crypto Officer**를 자동 부여

### 로컬 도구

- **Azure CLI** (Bicep 포함) — `az --version`에서 Bicep CLI가 표시되어야 함
- **Docker** — 로컬 빌드용 (선택: `az acr build`는 Docker 없이 클라우드 빌드 가능)
- **Python 3.12+** — `deploy.py`용
- **PowerShell** — `deploy-infra.ps1`용

---

## 단계별 배포

### Step 1: 엔터프라이즈 인프라 배포

```powershell
# Azure 로그인
az login

# 전체 배포 (VNET, Key Vault, Storage, AI Services + Foundry Project, AI Search, ACR)
cd enterprise
.\deploy-infra.ps1 -ResourceGroup "rg-hr-agent-enterprise" -Location "eastus2" -Prefix "hragent"
```

이 명령어는 **모든** Azure 리소스를 다음과 함께 생성합니다:

- ✅ 모든 서비스에 프라이빗 엔드포인트
- ✅ Storage, AI Services, ACR에 CMK 암호화 (+ AI Search에 CMK 적용)
- ✅ AI Services, AI Search, Storage에서 API 키 비활성화
- ✅ 관리 ID에 RBAC 역할 할당

스크립트가 다음 단계에 필요한 모든 값을 출력합니다.

### Step 1.5: 퍼블릭 접근 임시 활성화 (초기 설정용)

> **중요:** 모든 서비스는 기본적으로 퍼블릭 접근이 **비활성화**된 상태로 배포됩니다. 로컬 머신에서 컨테이너 이미지를 푸시하고 에이전트를 등록하려면 퍼블릭 접근을 임시로 활성화해야 합니다. 이것은 초기 설정의 예상된 워크플로우입니다 — Step 5에서 다시 잠급니다.
>
> 실제 엔터프라이즈 환경에서는 VNET 내부에서 수행합니다 (VPN, ExpressRoute, 또는 Azure Bastion이 있는 점프박스 VM). 이 데모/워크스루에서는 임시로 접근을 엽니다.

```powershell
# AI Services에 퍼블릭 접근 활성화
az cognitiveservices account update -g <rg> -n <ai-services-name> `
    --custom-domain <ai-services-name> `
    --set properties.publicNetworkAccess=Enabled

# ACR에 퍼블릭 접근 활성화
az acr update -n <acr-name> --public-network-enabled true

# AI Search에 퍼블릭 접근 활성화 (REST 사용 — CLI에 UserAssigned identity 버그 있음)
az rest --method PATCH `
    --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Search/searchServices/<search-name>?api-version=2024-06-01-preview" `
    --headers Content-Type=application/json `
    --body '{"properties":{"publicNetworkAccess":"enabled"}}'
```

### Step 2: 컨테이너 이미지 빌드 및 푸시

```powershell
# 옵션 A: 클라우드 빌드 (권장 — 프라이빗 엔드포인트에서도 작동, 로컬 Docker 불필요)
az acr build --registry <acr-name> --image hr-hosted-agent:latest --platform linux/amd64 .

# 옵션 B: 로컬 빌드 + 푸시 (Docker Desktop 실행 필요)
docker build --platform linux/amd64 -t <acr-login-server>/hr-hosted-agent:latest .
az acr login --name <acr-name>
docker push <acr-login-server>/hr-hosted-agent:latest
```

### Step 3: Foundry에 에이전트 등록

```powershell
# 환경 변수 설정 (Step 1 출력의 값 사용)
$env:AZURE_AI_PROJECT_ENDPOINT = "<project-endpoint-from-step-1>"
$env:AZURE_SEARCH_ENDPOINT = "<search-endpoint-from-step-1>"
$env:CONTAINER_IMAGE = "<acr-login-server>/hr-hosted-agent:latest"

# 배포
python deploy.py
```

### Step 4: 에이전트 시작

**Azure AI Foundry 포털** → **Agents** → 에이전트 찾기 → **Start**.

### Step 5: 퍼블릭 접근 재비활성화 (잠금)

> 에이전트가 배포되고 실행된 후, 퍼블릭 접근을 다시 비활성화합니다. 호스팅 에이전트는
> Foundry 인프라 **내부**에서 실행되며 프라이빗 엔드포인트를 통해 모든 서비스와 통신합니다
> — 퍼블릭 접근이 필요하지 않습니다.

```powershell
# AI Services 퍼블릭 접근 비활성화
az cognitiveservices account update -g <rg> -n <ai-services-name> `
    --custom-domain <ai-services-name> `
    --set properties.publicNetworkAccess=Disabled

# ACR 퍼블릭 접근 비활성화
az acr update -n <acr-name> --public-network-enabled false

# AI Search 퍼블릭 접근 비활성화
az rest --method PATCH `
    --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Search/searchServices/<search-name>?api-version=2024-06-01-preview" `
    --headers Content-Type=application/json `
    --body '{"properties":{"publicNetworkAccess":"disabled"}}'
```

이후에는 VNET 내부의 프라이빗 엔드포인트를 통해서만 이러한 서비스에 접근할 수 있습니다.
호스팅 에이전트는 Foundry가 프라이빗 네트워크를 통해 트래픽을 라우팅하므로 계속 작동합니다.

---

## Bicep 모듈 상세

### Managed Identity (`managed-identity.bicep`)

모든 리소스에서 사용하는 단일 **User-Assigned Managed Identity**를 생성합니다. 키리스 인증 전략의 핵심 — 하나의 ID, 모든 RBAC 역할:

| 역할 | 리소스 | 용도 |
| --- | --- | --- |
| Key Vault Crypto User | Key Vault | 암호화/복호화를 위한 CMK 키 사용 |
| Storage Blob Data Contributor | Storage | Blob 읽기/쓰기 |
| Cognitive Services OpenAI Contributor | AI Services | OpenAI 모델 호출 |
| Search Index Data Reader | AI Search | 검색 인덱스 쿼리 |
| Search Service Contributor | AI Search | 검색 서비스 관리 |
| AcrPull + AcrPush | Container Registry | 컨테이너 이미지 풀/푸시 |

### Network (`network.bicep`)

프라이빗 엔드포인트 서브넷과 **7개 프라이빗 DNS 존**이 있는 VNET을 생성합니다:

| DNS 존 | 서비스 |
| --- | --- |
| `privatelink.cognitiveservices.azure.com` | AI Services |
| `privatelink.openai.azure.com` | OpenAI 엔드포인트 |
| `privatelink.search.windows.net` | AI Search |
| `privatelink.blob.core.windows.net` | Storage (Blob) |
| `privatelink.file.core.windows.net` | Storage (File) |
| `privatelink.vaultcore.azure.net` | Key Vault |
| `privatelink.azurecr.io` | Container Registry |

각 DNS 존은 VNET에 연결되어 프라이빗 엔드포인트 DNS 확인이 자동으로 작동합니다.

### Key Vault (`keyvault.bicep`)

- **RBAC 전용** (`enableRbacAuthorization: true`) — 접근 정책 없음
- **제거 보호** 활성화 (CMK에 필요)
- 다른 모든 서비스가 사용하는 **RSA-2048 CMK 키** 생성
- 방화벽: `defaultAction: Deny`, `bypass: AzureServices`

### AI Services (`ai-services.bicep`)

- **`disableLocalAuth: true`** — API 키 완전 비활성화
- **`allowProjectManagement: true`** — 하위 리소스로 Foundry 프로젝트 생성 가능
- User-assigned MI로 CMK 암호화
- OpenAI 모델 배포 (기본값: `gpt-4.1`)
- `cognitiveservices`와 `openai` DNS 존 모두에 프라이빗 엔드포인트 생성

### AI Search (`ai-search.bicep`)

- **`disableLocalAuth: true`** — API 키나 쿼리 키 없음
- **`encryptionWithCmk.enforcement: 'Enabled'`** — 모든 새 인덱스가 CMK를 사용해야 함
- 시맨틱 검색 활성화 (지식 기반 그라운딩용)

> **참고:** CMK 적용은 서비스 수준입니다. 실제 CMK 키는 SDK를 통해 인덱스 생성 시 구성해야 합니다. [Azure AI Search CMK 문서](https://learn.microsoft.com/ko-kr/azure/search/search-security-manage-encryption-keys) 참조.

### ACR (`acr.bicep`)

- **Premium SKU** (CMK + 프라이빗 엔드포인트에 필요)
- **관리자 사용자 비활성화** — RBAC로만 풀/푸시
- 비버전 키 URI를 사용한 CMK 암호화 (자동 로테이션 호환)

---

## 초기 설정 vs. 프로덕션 (프라이빗 엔드포인트 워크플로우)

모든 서비스에서 `publicNetworkAccess: 'Disabled'`인 경우, 퍼블릭 인터넷에서 접근할 수 없습니다 — 여기에는 노트북과 Azure AI Foundry 포털이 포함됩니다.

**초기 설정** (이미지 푸시, 에이전트 배포, 포털에서 확인) 시 세 가지 옵션이 있습니다:

1. **퍼블릭 접근 임시 활성화** (가장 간단 — 이 워크스루에서 사용)
   - 퍼블릭 접근 열기 → 이미지 푸시 → 에이전트 배포 → 확인 → 재비활성화
   - 위의 배포 가이드 Step 1.5와 Step 5 참조

2. **VNET 내부의 점프박스 VM 사용** (엔터프라이즈 표준)
   - VNET에 VM + Azure Bastion 배포
   - VM에 RDP/SSH하여 `az acr build`, `python deploy.py` 실행 및 포털 접근

3. **VPN으로 연결** (기존 인프라가 있는 엔터프라이즈)
   - Point-to-Site VPN 게이트웨이 (~30분 설정)
   - Site-to-Site VPN 또는 ExpressRoute (회사 네트워크가 이미 연결됨)

**에이전트가 배포된 후**, Foundry의 관리 인프라 내부에서 실행되며 **프라이빗 엔드포인트**를 통해 모든 서비스와 통신합니다 — 퍼블릭 접근이 필요하지 않습니다. 관리 작업에만 퍼블릭 접근 (또는 VNET 접근)이 필요합니다.

---

## 문제 해결

### 첫 실행 시 배포 실패

RBAC 역할 할당이 전파되는 데 5-10분이 걸릴 수 있습니다. 권한 오류 (특히 Key Vault 키 생성 또는 CMK 구성 관련)로 배포가 실패하면 5분 기다린 후 재시도하세요.

### ACR 푸시 실패

ACR이 `publicNetworkAccess: 'Disabled'` 상태입니다. 이미지를 푸시하려면:

- `az acr build`를 사용하여 클라우드에서 빌드 (권장 — 프라이빗 엔드포인트에서도 작동)
- 또는 퍼블릭 접근 임시 활성화: `az acr update --name <acr> --public-network-enabled true`

### Foundry 포털에 "연결할 수 없음" 표시

퍼블릭 접근이 비활성화된 경우 예상되는 동작입니다. VNET 내부에서 (VM/VPN을 통해) 포털에 접근하거나 AI Services에 퍼블릭 접근을 임시로 활성화하세요.

### 모델 배포 불가

`gpt-4.1` 모델이 모든 리전에서 사용 가능하지 않을 수 있습니다. [모델 가용성](https://learn.microsoft.com/ko-kr/azure/ai-services/openai/concepts/models)을 확인하고 `modelDeploymentName` 파라미터를 조정하세요.

---

## 보안 체크리스트

배포 후 (퍼블릭 접근 재비활성화 후) 확인사항:

- [ ] 모든 서비스의 네트워킹 설정에 "프라이빗 엔드포인트" 표시
- [ ] AI Services → 키 및 엔드포인트 → "로컬 인증: 비활성화"
- [ ] AI Search → 키 → API 키 없음
- [ ] Storage → 구성 → "스토리지 계정 키 접근 허용: 비활성화"
- [ ] Key Vault → 접근 구성 → "권한 모델: Azure 역할 기반 접근 제어"
- [ ] ACR → 접근 키 → "관리자 사용자: 비활성화"
- [ ] 모든 서비스에 퍼블릭 IP 없음

포함된 `validate-enterprise.ps1` 스크립트로 자동화할 수 있습니다:

```powershell
.\validate-enterprise.ps1 -ResourceGroup "rg-hr-agent-enterprise"
```

---

## 의존성

Part 1과 동일 — 앱 코드는 변경되지 않습니다:

| 패키지 | 용도 |
| --- | --- |
| `azure-ai-agentserver-agentframework` | 호스팅 어댑터 |
| `agent-framework-core` | 코어 Agent Framework |
| `agent-framework-azure-ai` | Azure AI 클라이언트 |
| `agent-framework-azure-ai-search` | AI Search 컨텍스트 프로바이더 |
| `azure-ai-projects` | Foundry SDK (deploy.py) |
| `azure-identity` | Azure 인증 (DefaultAzureCredential) |
