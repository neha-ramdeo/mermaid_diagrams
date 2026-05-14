# Financebot — Threat Model View

Same architecture, annotated with trust zones, control state, and the top threats from the security review.

Legend: ✅ control present · ⚠️ weak / partial · ❌ missing · ❓ unknown

```mermaid
flowchart LR
  %% ===================== TRUST ZONES =====================
  subgraph ZUser["🌐 Untrusted: User Devices"]
    Web["financebot-ux<br/>Next.js (Vercel)<br/>Token: React Query in-memory"]
    Mobile["mobile app<br/>React Native<br/>Token: in-memory · ❌ no cert pinning<br/>❌ no jailbreak/root detection"]
  end

  subgraph ZEdge["🟡 Edge"]
    CF["Cloudflare"]
  end

  subgraph ZApp["🟢 App tier (internal network)"]
    Rails["financebot – Rails<br/>✅ bearer auth on HTTP<br/>❌ gRPC server has NO auth<br/>⚠️ AccessControl#privileged? = TODO (returns true)"]
    Py["financebot-ai – Python<br/>✅ JWT verify<br/>⚠️ introspection optional<br/>⚠️ prompt injection via detection_context<br/>⚠️ tools execute w/o per-call authz"]
    RailsDB[("Rails DB<br/>Entity · PlaidAccount<br/>DataRetrieval audit")]
    PyDB[("Python DB<br/>Conversations · Messages<br/>🔒 KMS-encrypted")]
    Memory["Short/Long-term memory<br/>⚠️ Bedrock isolation unverified"]
  end

  subgraph ZIdent["🔵 Identity"]
    Keycloak["Keycloak"]
    UW["UW monolith"]
  end

  subgraph ZSensitive["🔴 Sensitive internal services"]
    LegalConsent["Legal Consent (gRPC)"]
    SpiceDB["SpiceDB<br/>✅ consent gate<br/>❌ not used for resource authz"]
    ConsumerProfile["Consumer Profile (gRPC)<br/>📋 PII"]
    CBMS["CBMS (HTTPS)<br/>💳 Credit reports<br/>❓ auth method unknown"]
  end

  subgraph ZExt["⚫ External"]
    Plaid["Plaid<br/>✅ webhook signature verified"]
    LLM["LLM provider<br/>via LiteLLM<br/>⚠️ Plaid balances sent in plaintext"]
  end

  %% ===================== FLOWS =====================
  Web -->|login| Keycloak
  Mobile -->|login| Keycloak
  UW <-->|token exchange| Web
  UW <-->|token exchange| Mobile
  Keycloak <--> UW

  Web -->|"Bearer (HTTPS)"| CF
  Mobile -->|"Bearer (HTTPS · no pinning)"| CF
  CF --> Rails
  CF --> Py

  Py ===>|"gRPC · UNAUTHENTICATED ⚠️"| Rails
  Rails <-->|events| Py
  Py --> Memory
  Py --> PyDB
  Rails --> RailsDB

  Rails -->|"consent check"| SpiceDB
  Rails -->|"record consent"| LegalConsent
  LegalConsent -->|store| SpiceDB
  Rails -->|"PII fetch"| ConsumerProfile
  Rails -->|"credit pull"| CBMS
  Rails -->|"link accounts"| Plaid
  Plaid -.->|webhook ✅ sig verified| Rails

  Py -.->|"prompt + Plaid data ⚠️"| LLM

  %% ===================== THREATS =====================
  T1["🔥 T1: Unauthenticated gRPC<br/>→ cross-user credit/Plaid exfil"]:::threat
  T2["🔥 T2: Stub provider authz<br/>→ stolen token = full data access"]:::threat
  T3["🔥 T3: Prompt injection<br/>→ tool abuse + phishing via markdown"]:::threat

  T1 -.-> Rails
  T2 -.-> Rails
  T3 -.-> Py
  T3 -.-> Web
  T3 -.-> Mobile

  %% ===================== STYLES =====================
  classDef threat fill:#fee2e2,stroke:#b91c1c,color:#7f1d1d,font-weight:bold
  classDef user fill:#fde2e4,stroke:#c53030,color:#000
  classDef edge fill:#fef3c7,stroke:#b45309,color:#000
  classDef app fill:#bdf0d4,stroke:#2f855a,color:#000
  classDef ident fill:#dbeafe,stroke:#1d4ed8,color:#000
  classDef sens fill:#fecaca,stroke:#b91c1c,color:#000
  classDef ext fill:#e2e8f0,stroke:#4a5568,color:#000

  class Web,Mobile user
  class CF edge
  class Rails,Py,RailsDB,PyDB,Memory app
  class Keycloak,UW ident
  class LegalConsent,SpiceDB,ConsumerProfile,CBMS sens
  class Plaid,LLM ext
```

## Top threats

1. **Unauthenticated inbound gRPC on the Rails service** — `bin/grpc_server` disables default interceptors; any caller on the internal network can fetch credit reports / Plaid data for arbitrary `entity_uuid`.
2. **Stubbed provider-level authorization** — `Authorization::AccessControl#privileged?` returns `true`; a stolen token has full provider access for its bound entity, with no record-level defense-in-depth.
3. **Prompt injection via `detection_context`** — unescaped user input lands in the LLM system prompt; tools execute without per-call authorization; unsanitized markdown rendering on web + mobile turns the assistant output into a phishing channel.
