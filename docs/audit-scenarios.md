# Audit Scenarios — Configuration Guide

How to configure your load balancers for three common audit scenarios.

---

## Scenario 1: Standard Audit (No Configuration Required)

Every LB is compared against the full security baseline. Service policies are compared against the default namespace.

```mermaid
flowchart LR
    subgraph XC["XC Console"]
        LB["HTTP Load Balancer\n(any namespace)"]
        Baseline["Security Baseline\n(extension settings)"]
        DefaultNS["Default Namespace\nService Policies"]
    end

    subgraph Audit["Audit Result"]
        Compare["Compare every setting\nagainst baseline"]
        Result["Per-LB Report Card"]
    end

    LB --> Compare
    Baseline --> Compare
    DefaultNS -->|"policy comparison"| Compare
    Compare --> Result

    style XC fill:#e8f4f8,stroke:#2196F3
    style Audit fill:#e8f5e9,stroke:#4CAF50
```

### What You See in the Report

```mermaid
flowchart TD
    subgraph Report["Report Output"]
        direction TB
        LB1["✅ app-compliant — 12 passed, 0 warnings"]
        LB2["⚠️ app-basic — 8 passed, 4 warnings"]

        subgraph Warnings["Expanded Warnings"]
            W1["⚠️ TLS — missing"]
            W2["⚠️ HSTS — missing"]
            W3["⚠️ WAF Blocking Mode — expected: Block, found: Monitor"]
            W4["⚠️ Bot Defense — missing"]
        end

        subgraph Passed["Expanded Passed"]
            P1["✅ App Firewall"]
            P2["✅ Geo Policy"]
            P3["✅ IP Reputation"]
        end
    end

    LB2 --> Warnings
    LB1 --> Passed

    style Report fill:#fff,stroke:#ddd
    style Warnings fill:#fff3e0,stroke:#FF9800
    style Passed fill:#e8f5e9,stroke:#4CAF50
```

### Configuration Steps

**None required.** Install the extension or configure the webapp, navigate to the LB list page, and the audit runs automatically.

---

## Scenario 2: Baseline LB Override

A reference LB in the **default** namespace serves as a "known-good" configuration. Other LBs are compared against it field by field — a gap the reference LB shares on the exact same setting is suppressed, and a check the target passes but the reference fails is flagged as "not in baseline" (which turns the target's badge to ⚠️).

```mermaid
flowchart LR
    subgraph XC["XC Console — Default Namespace"]
        RefLB["🏠 baseline-standard\n(reference LB)"]
    end

    subgraph Target["XC Console — Production Namespace"]
        TargetLB["📋 app-production\nLabel: xc-audit-baseline-lb = baseline-standard"]
    end

    subgraph Audit["Audit Comparison"]
        direction TB
        BothFail["Both fail the same\nDDoS setting?\n→ Suppressed (via baseline)"]
        OnlyTarget["Only target fails WAF?\n→ ⚠️ Warning"]
        OnlyBaseline["Target passes but\nbaseline fails API Discovery?\n→ ⚠️ Not in baseline"]
    end

    RefLB -->|"audited against\nsame baseline"| Audit
    TargetLB -->|"compared to\nreference LB"| Audit

    style XC fill:#e8f4f8,stroke:#2196F3
    style Target fill:#fff3e0,stroke:#FF9800
    style Audit fill:#f3e8f9,stroke:#9C27B0
```

### What You See in the Report

```mermaid
flowchart TD
    subgraph Report["Report Output for app-production"]
        direction TB
        Header["⚠️ app-production  [ref: baseline-standard]\n(8 passed, 1 warnings, 1 not in baseline)"]

        subgraph Active["Active Warnings"]
            W1["⚠️ WAF Blocking Mode — expected: Block, found: Monitor"]
        end

        subgraph Via["Suppressed — counted in passed"]
            V1["✅ DDoS Mitigation Rules — via baseline-standard"]
            V2["✅ L7 DDoS Protection — via baseline-standard"]
        end

        subgraph NIB["Not in Baseline — sets LB to ⚠️"]
            N1["ℹ️ API Discovery — not in baseline"]
        end
    end

    Header --> Active
    Header --> Via
    Header --> NIB

    style Report fill:#fff,stroke:#ddd
    style Active fill:#fff3e0,stroke:#FF9800
    style Via fill:#e8f5e9,stroke:#4CAF50
    style NIB fill:#e3f2fd,stroke:#2196F3
```

### Configuration Steps

#### Step 1 — Create the Reference LB

Create an HTTP Load Balancer in the **default** namespace that represents your approved configuration (e.g., `baseline-standard`). This LB doesn't need to serve traffic — it's just a configuration template.

#### Step 2 — Register the Audit Labels (once per tenant)

XC only accepts label keys that are registered as known labels. With an XC console tab open, go to the extension's **Options → Exemption Map** and click **Register Labels in XC**. This registers every `xc-audit-*` key in the `shared` namespace. Skip this step if it has already been done for the tenant.

#### Step 3 — Add the Label to Target LBs

In the XC Console, navigate to your target LB:

1. **Load Balancers** → select the LB → **Edit**
2. Scroll to **Labels**
3. Add a label:
   - **Key:** `xc-audit-baseline-lb`
   - **Value:** the name of your reference LB (e.g., `baseline-standard`)
4. **Save and Exit**

The value must exactly match the name of an LB in the **default** namespace. If it doesn't (a typo, or the reference LB is in another namespace or was deleted), the label is silently ignored and the LB gets a standard audit. To confirm the override took effect, check that the LB's badge shows a **ref: baseline-standard** tag.

```
┌─────────────────────────────────────────────────┐
│  Edit HTTP Load Balancer: app-production        │
│                                                 │
│  Labels                                         │
│  ┌───────────────────────┬────────────────────┐ │
│  │ Key                   │ Value              │ │
│  ├───────────────────────┼────────────────────┤ │
│  │ xc-audit-baseline-lb  │ baseline-standard  │ │
│  └───────────────────────┴────────────────────┘ │
│  + Add Label                                    │
│                                                 │
│            [Save and Exit]  [Cancel]            │
└─────────────────────────────────────────────────┘
```

#### How It Works

Comparison is by **setting**, not by check. Each failure is identified by the exact field that differs from the security baseline (e.g., `spec.l7_ddos_protection.mitigation_block`), and the target's failure is only suppressed when the reference LB fails on that same field. Inspector checks (WAF blocking, IP reputation, geo policy, …) are compared per inspector.

| Current LB | Baseline LB | Result |
|:---:|:---:|---|
| ⚠️ Fails setting X | ⚠️ Fails setting X | **Suppressed** — counted as passed, shown as "via baseline-standard" |
| ⚠️ Fails setting X | ✅ Passes, or fails a different setting | **Warning** — LB differs from baseline |
| ✅ Passes setting X | ⚠️ Fails setting X | **Not in baseline** — shown as "not in baseline" and sets the LB to ⚠️ |
| ✅ Passes | ✅ Passes | **Passed** — both comply |

Because matching is per setting, a check with several sub-settings can land in more than one row. For example, if the reference LB's L7 DDoS protection uses a JS challenge instead of blocking, and the target's uses a custom DDoS policy, the two failures are on different fields and the target still gets a warning. (An LB with no L7 DDoS block set uses XC's platform default, which blocks, so it passes this check.) To clear a "not in baseline" result, bring the reference LB up to the target's level.

---

## Scenario 3: Selective Ignore Labels

Skip specific checks for individual LBs using exemption labels. Useful for LBs that intentionally don't need certain features (e.g., a static site that doesn't need Bot Defense).

```mermaid
flowchart LR
    subgraph XC["XC Console"]
        LB["📋 marketing-site\nLabels:\n  xc-audit-ignore-bot = true\n  xc-audit-ignore-apid = true"]
    end

    subgraph Audit["Audit Processing"]
        direction TB
        Full["Full Baseline\n(all check settings)"]
        Filter["Remove settings mapped to\nignore-bot and ignore-apid"]
        Reduced["Filtered Baseline"]
    end

    subgraph Result["Report Output"]
        Active["Active Checks\n8 passed, 2 warnings"]
        Skipped["Skipped Checks\nBot Defense — Ignored by Label\nAPI Discovery — Ignored by Label"]
    end

    LB --> Full
    Full --> Filter
    Filter --> Reduced
    Reduced --> Active
    Filter --> Skipped

    style XC fill:#e8f4f8,stroke:#2196F3
    style Audit fill:#fff3e0,stroke:#FF9800
    style Result fill:#e8f5e9,stroke:#4CAF50
```

### What You See in the Report

```mermaid
flowchart TD
    subgraph Report["Report Output for marketing-site"]
        direction TB
        Header["⚠️ marketing-site\n(8 passed, 2 warnings, 2 skipped)"]

        subgraph Warnings["Active Warnings"]
            W1["⚠️ TLS — missing"]
            W2["⚠️ HSTS — missing"]
        end

        subgraph Pass["Passed"]
            P1["✅ App Firewall"]
            P2["✅ WAF Blocking Mode"]
            P3["✅ Geo Policy"]
        end

        subgraph Skip["Skipped (Ignored by Label)"]
            S1["Bot Defense — Ignored by Label"]
            S2["API Discovery — Ignored by Label"]
        end
    end

    Header --> Warnings
    Header --> Pass
    Header --> Skip

    style Report fill:#fff,stroke:#ddd
    style Warnings fill:#fff3e0,stroke:#FF9800
    style Pass fill:#e8f5e9,stroke:#4CAF50
    style Skip fill:#f5f5f5,stroke:#999
```

### Configuration Steps

#### Step 1 — Choose Which Checks to Skip

| Label Key | Skips | Category |
|-----------|-------|----------|
| `xc-audit-ignore-tls` | TLS configuration | Core Security |
| `xc-audit-ignore-hsts` | HSTS headers | Core Security |
| `xc-audit-ignore-iprep` | IP Reputation checks | Core Security |
| `xc-audit-ignore-geo` | Geo Policy checks | Core Security |
| `xc-audit-ignore-trustip` | Trust Client IP Headers | Core Security |
| `xc-audit-ignore-waf` | App Firewall + all WAF inspections | WAF |
| `xc-audit-ignore-mp` | Malware Protection | WAF |
| `xc-audit-ignore-ddos` | DDoS Mitigation + L7 DDoS | DDoS |
| `xc-audit-ignore-bot` | Bot Defense | Bot Defense |
| `xc-audit-ignore-csd` | Client-Side Defense | Bot Defense |
| `xc-audit-ignore-apid` | API Discovery | API Security |
| `xc-audit-ignore-apip` | API Protection + API Testing | API Security |
| `xc-audit-ignore-jwt` | JWT Validation | API Security |
| `xc-audit-ignore-sp` | Service Policies | Policy & Data |
| `xc-audit-ignore-sdp` | Sensitive Data Policy | Policy & Data |

Some labels cover more than one check. `xc-audit-ignore-waf` alone skips 5 (App Firewall plus four WAF inspections), and `xc-audit-ignore-ddos` and `xc-audit-ignore-apip` skip 2 each. Each skipped check is counted separately in the badge's "skipped" total.

#### Step 2 — Register the Audit Labels (once per tenant)

XC only accepts label keys that are registered as known labels. With an XC console tab open, go to the extension's **Options → Exemption Map** and click **Register Labels in XC**. Skip this step if it has already been done for the tenant.

#### Step 3 — Add Labels to the LB

In the XC Console, navigate to your target LB:

1. **Load Balancers** → select the LB → **Edit**
2. Scroll to **Labels**
3. Add one label per check you want to skip:
   - **Key:** `xc-audit-ignore-bot` (from table above)
   - **Value:** `true`
4. **Save and Exit**

```
┌─────────────────────────────────────────────────┐
│  Edit HTTP Load Balancer: marketing-site        │
│                                                 │
│  Labels                                         │
│  ┌───────────────────────┬────────────────────┐ │
│  │ Key                   │ Value              │ │
│  ├───────────────────────┼────────────────────┤ │
│  │ xc-audit-ignore-bot   │ true               │ │
│  ├───────────────────────┼────────────────────┤ │
│  │ xc-audit-ignore-apid  │ true               │ │
│  └───────────────────────┴────────────────────┘ │
│  + Add Label                                    │
│                                                 │
│            [Save and Exit]  [Cancel]            │
└─────────────────────────────────────────────────┘
```

> **Note:** Labels must use the exact keys from the table. The value must be `true`. Custom exemption codes can be defined in the extension's Options page under **Exemption Map**.
