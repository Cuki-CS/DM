# Salesforce Deployment Flow (Dev → UAT → Prod + Hotfix)

## 🧩 Full Architecture

```mermaid
flowchart LR

subgraph Git["Git Branches"]
A[feature/*] --> B[dev]
B --> C[uat]
C --> D[master]
end

subgraph CI["GitHub Actions"]
E[Validate PR]
F[Deploy DEV]
G[Deploy UAT]
H[Deploy PROD]
end

subgraph SF["Salesforce Environments"]
I[DEV Org]
J[UAT Org - Integration Testing]
K[PROD Org]
end

B --> E
E --> F
F --> I
I --> G
G --> J
J --> H
H --> K

%% Hotfix Flow
D --> L[hotfix/*]
L --> M[Deploy to UAT]
M --> J
J --> N[Integration Validation]
N --> O[Merge to master]
O --> P[Deploy to PROD]
P --> K
O --> Q[Back-merge to dev]
Q --> B
```

---

## 🔁 Normal Release Flow

```mermaid
flowchart LR

A[feature branch] --> B[PR to dev]
B --> C[Validate PR]
C --> D[Deploy to DEV]
D --> E[DEV Testing]

E --> F[Merge dev to uat]
F --> G[Deploy to UAT]
G --> H[UAT Integration and Signoff]

H --> I[Merge uat to master]
I --> J[Deploy to PROD]
J --> K[Post-release validation]
```

---

## 🚑 Hotfix Flow (via UAT)

```mermaid
flowchart LR

A[Production Issue] --> B[Create hotfix from master]
B --> C[Validate hotfix]

C --> D[Deploy to UAT]
D --> E[Integration Testing]

E --> F{Approved}

F -->|Yes| G[Merge to master]
G --> H[Deploy to PROD]
H --> I[Fix live]

G --> J[Back-merge to dev]

F -->|No| X[Fix and retest]
X --> D
```

---

## ⚖️ GitHub Actions vs Salesforce

```mermaid
flowchart TB

subgraph Git
A[Branches control flow]
end

subgraph Actions["GitHub Actions"]
B[Run tests]
C[Code scan]
D[Build package]
E[Authenticate]
F[Deploy command]
end

subgraph Salesforce
G[Metadata deployed]
H[Org updated]
I[Tests executed]
J[User validation]
end

A --> B --> C --> D --> E --> F --> G --> H --> I --> J
```

---

## 🧱 Branch Strategy

```mermaid
flowchart LR

A[feature/*] --> B[dev]
B --> C[uat]
C --> D[master]
D --> E[PROD]

D --> F[hotfix/*]
F --> C
C --> D
F --> B
```
