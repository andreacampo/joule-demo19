# Operations Score — Computation Model

## Overview

The Operations Score is computed daily per managed system (identified by `lmsId`). Raw operational metrics are collected from **SAP Cloud ALM Analytics** APIs, transformed into per-metric compliance values (0–100), aggregated into **KPI scores**, then into **Area Scores**, and finally composed into the single **Operations Score**.

---

## Step 1 — Raw Metric Collection

Raw metric values (`metricRawValue`) are ingested from SAP Cloud ALM Analytics. Each metric belongs to exactly one KPI and one of three area score domains:

- **SYSTEM STATUS** (System Health area) — performance/availability/exceptions/background jobs
- **OPERATION SETUP** (Operations Setup area) — monitoring configuration status
- **SECURITY SCORE** (Security Score area) — security check outcomes

---

## Step 2 — Metric Compliance Calculation

Each metric produces a **metric compliance value** in the range [0, 100].

### 2a. SYSTEM_STATUS_HEALTH metrics — Logistic (Sigmoid) Function

Used for: all SYSTEM STATUS metrics (response time, heartbeat, errors, dumps, failed jobs).

**Current formula** (feature flag `FEATURE_FLAG_COMPLIANCY_CALC` = enabled):

```
compliance = (1 / (1 + exp(−k × (x − x₀)))) × 100
```

**Legacy formula** (feature flag disabled; range [−100, 100], deprecated):

```
compliance = (2 / (1 + exp(−k × (x − x₀))) − 1) × 100
```

Parameters:
- `x` = current `metricRawValue`
- `x₀` = `slaRawValue` (the SLA threshold — the raw value that produces 50% compliance)
- `k` = `logisticGrowthRate` (negative for LOW-type metrics, positive for HIGH-type)

**SLA type**:
- `LOW` → higher raw value is worse → k is negative (compliance decreases as x rises)
- `HIGH` → higher raw value is better → k is positive (compliance increases as x rises)

**At the SLA point** (x = x₀): compliance = 50%

### 2b. OPERATION_SETUP metrics — Binary

Used for: all Operations Setup metrics (configuration and runtime status checks).

```
compliance = rawValue == threshold ? 100 : 0
```

- `CONFIGURATION_STATUS` type: threshold = 2.0
- `RUNTIME_STATUS` type: threshold = 1.0

### 2c. SECURITY_SCORE metrics — Binary

Used for: all Security Score metrics (security checks).

```
compliance = rawValue == 1.0 ? 100 : 0
```

A value of 1.0 means the check passed; anything else means the check failed.

### Metric Trend Color

After compliance is computed, a trend color is assigned per metric:
- **GREEN**: compliance > 85
- **YELLOW**: 70 ≤ compliance ≤ 85
- **RED**: compliance < 70

---

## Step 3 — KPI Score

For each KPI, the score is the weighted sum of its constituent metric compliance values:

```
KPI_score = Σ max(0, metric_compliance_i × metric_weight_i / 100)
```

`metric_weight_i` is from `MetricSLAComplianceConfig`. Weights within a KPI sum to 100%.

---

## Step 4 — Area Scores

Each area score is the weighted sum of its KPI scores:

```
AreaScore = Σ (KPI_score_j × kpi_weight_j / 100)
```

`kpi_weight_j` is from `KPISLAComplianceConfig`. Weights within an area sum to 100%.

---

## Step 5 — Operations Score

```
OperationsScore = 0.4 × SecurityScore + 0.3 × SystemHealth + 0.3 × OperationsSetup
```

Rounded to 1 decimal place.

**Trend color** thresholds apply to the Operations Score and all Area Scores:
- **GREEN**: score > 85
- **YELLOW**: 70 ≤ score ≤ 85
- **RED**: score < 70

---

## Metric / KPI / Weight Reference

### SYSTEM STATUS → System Health Area Score

#### KPIs and weights within System Health:

| kpiId | kpiWeight |
|-------|-----------|
| AVAILABILITY | 50 |
| EXCEPTIONS | 25 |
| BACKGROUND | 15 |
| PERFORMANCE | 10 |

#### Metrics:

| metricId | kpiId | metricWeight | slaRawValue | logisticGrowthRate | slaType | notes |
|----------|-------|--------------|-------------|-------------------|---------|-------|
| ABAP_SYSTEM_HEARTBEAT | AVAILABILITY | 90 | 10 | -1.8 | HIGH | |
| ABAP_INSTANCE_HEARTBEAT | AVAILABILITY | 10 | 10 | -1.8 | HIGH | |
| NO_OF_DUMPS | EXCEPTIONS | 70 | 400 | -0.05 | LOW | Exception Monitoring; daily total |
| NO_OF_SYSLOG_ERRORS | EXCEPTIONS | 25 | 9000 | -0.002 | LOW | Exception Monitoring; daily total |
| NO_OF_APPLICATION_LOG_ERRORS | EXCEPTIONS | 5 | 2500 | -0.004 | LOW | Exception Monitoring; daily total |
| FAILED_JOBS | BACKGROUND | 100 | 2.24 | -5 | LOW | Health Monitoring; avg of 288 daily 5-min readings |
| ABAP_RESPONSE_TIME_AVG | PERFORMANCE | 90 | 4000 | -0.025 | LOW | |
| ABAP_RESPONSE_TIME_MAX | PERFORMANCE | 10 | 200000 | -0.003 | LOW | |

---

### OPERATION SETUP → Operations Setup Area Score

#### KPIs and weights within Operations Setup:

| kpiId | kpiWeight |
|-------|-----------|
| HEALTH_MONITORING_SETUP | 20 |
| EXCEPTION_MONITORING_SETUP | 20 |
| INTEGRATION_MONITORING_SETUP | 15 |
| JOB_MONITORING_SETUP | 15 |
| CONFIGURATION_AND_SECURITY_ANALYSIS_SETUP | 15 |
| REAL_USER_MONITORING_SETUP | 5 |
| BUSINESS_SERVICE_MANAGEMENT | 5 |
| BUSINESS_PROCESS_MONITORING_SETUP | 5 |

#### Metrics (all binary, CONFIGURATION_STATUS unless noted):

| metricId | kpiId | metricWeight | statusType |
|----------|-------|--------------|------------|
| HEALTH_MONITORING_CONFIGURATION_STATUS | HEALTH_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| HEALTH_MONITORING_RUNTIME_STATUS | HEALTH_MONITORING_SETUP | 50 | RUNTIME_STATUS |
| EXCEPTION_MONITORING_CONFIGURATION_STATUS | EXCEPTION_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| EXCEPTION_MONITORING_RUNTIME_STATUS | EXCEPTION_MONITORING_SETUP | 50 | RUNTIME_STATUS |
| INTEGRATION_MONITORING_CONFIGURATION_STATUS | INTEGRATION_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| INTEGRATION_MONITORING_RUNTIME_STATUS | INTEGRATION_MONITORING_SETUP | 50 | RUNTIME_STATUS |
| JOB_MONITORING_CONFIGURATION_STATUS | JOB_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| JOB_MONITORING_RUNTIME_STATUS | JOB_MONITORING_SETUP | 50 | RUNTIME_STATUS |
| CONFIGURATION_AND_SECURITY_ANALYSIS_CONFIGURATION_STATUS | CONFIGURATION_AND_SECURITY_ANALYSIS_SETUP | 50 | CONFIGURATION_STATUS |
| CONFIGURATION_AND_SECURITY_ANALYSIS_RUNTIME_STATUS | CONFIGURATION_AND_SECURITY_ANALYSIS_SETUP | 50 | RUNTIME_STATUS |
| REAL_USER_MONITORING_CONFIGURATION_STATUS | REAL_USER_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| REAL_USER_MONITORING_RUNTIME_STATUS | REAL_USER_MONITORING_SETUP | 50 | RUNTIME_STATUS |
| BUSINESS_SERVICE_MANAGEMENT_CONFIGURATION_STATUS | BUSINESS_SERVICE_MANAGEMENT | 50 | CONFIGURATION_STATUS |
| BUSINESS_SERVICE_MANAGEMENT_RUNTIME_STATUS | BUSINESS_SERVICE_MANAGEMENT | 50 | RUNTIME_STATUS |
| BUSINESS_PROCESS_MONITORING_CONFIGURATION_STATUS | BUSINESS_PROCESS_MONITORING_SETUP | 50 | CONFIGURATION_STATUS |
| BUSINESS_PROCESS_MONITORING_RUNTIME_STATUS | BUSINESS_PROCESS_MONITORING_SETUP | 50 | RUNTIME_STATUS |

---

### SECURITY SCORE → Security Score Area Score

#### KPIs and weights within Security Score:

| kpiId | kpiWeight |
|-------|-----------|
| SYSTEM_HARDENING | 40 |
| VULNERABILITY_AND_LIFECYCLE | 40 |
| IDENTITY_AND_ACCESS_CONTROL | 20 |

#### Metrics — SYSTEM_HARDENING (all binary, weight ~5.9% each):

| metricId | kpiId | metricWeight |
|----------|-------|--------------|
| ABAP_MESSAGE_SERVER_ACCESS_CONTROL | SYSTEM_HARDENING | 5.9 |
| ACTIVATION_OF_SECURE_NETWORK_COMMUNICATIONS | SYSTEM_HARDENING | 5.9 |
| CLIENT_CONFIGURATION | SYSTEM_HARDENING | 5.9 |
| DATABASE_CLIENT_FILE_ACCESS | SYSTEM_HARDENING | 5.9 |
| DISABLED_ICFA_SERVICES | SYSTEM_HARDENING | 5.9 |
| DISABLED_ICFB_SERVICES | SYSTEM_HARDENING | 5.9 |
| DISABLED_ICF_SERVICES | SYSTEM_HARDENING | 5.9 |
| GATEWAY_SECURITY_SETTINGS | SYSTEM_HARDENING | 5.6 |
| HTTPS_ENFORCEMENT | SYSTEM_HARDENING | 5.9 |
| PROFILE_PARAMETER_FOR_DEFAULT_PASSWORDS | SYSTEM_HARDENING | 5.9 |
| PROFILE_PARAMETER_FOR_LOGON_SECURITY | SYSTEM_HARDENING | 5.9 |
| PROFILE_PARAMETER_FOR_PASSWORD_SECURITY | SYSTEM_HARDENING | 5.9 |
| RFC_GATEWAY_SECURITY | SYSTEM_HARDENING | 5.9 |
| RESTRICTION_OF_OS_COMMANDS | SYSTEM_HARDENING | 5.9 |
| SETTINGS_FOR_SNC_PROTECTION | SYSTEM_HARDENING | 5.9 |
| TABLE_ACCESS_PROTECTION | SYSTEM_HARDENING | 5.9 |
| UNNECESSARY_STANDARD_PROGRAMS | SYSTEM_HARDENING | 5.9 |

#### Metrics — VULNERABILITY_AND_LIFECYCLE (all binary, weight 25% each):

| metricId | kpiId | metricWeight |
|----------|-------|--------------|
| SUPPORT_PACKAGE_UPDATES | VULNERABILITY_AND_LIFECYCLE | 25 |
| AGE_OF_SAP_CRYPTOLIB_ASSEMBLY | VULNERABILITY_AND_LIFECYCLE | 25 |
| AGE_OF_KERNEL_COMPILE | VULNERABILITY_AND_LIFECYCLE | 25 |
| AGE_OF_SUPPORT_PACKAGES | VULNERABILITY_AND_LIFECYCLE | 25 |

#### Metrics — IDENTITY_AND_ACCESS_CONTROL (all binary):

| metricId | kpiId | metricWeight | notes |
|----------|-------|--------------|-------|
| SAPCPIC_USER | IDENTITY_AND_ACCESS_CONTROL | 14.3 | |
| DEFAULT_PASSWORD_OF_TMSADM | IDENTITY_AND_ACCESS_CONTROL | 14.3 | |
| TMSADM_IN_PRODUCTION_CLIENT | IDENTITY_AND_ACCESS_CONTROL | 14.3 | |
| EARLYWATCH_IN_OBSOLETE_CLIENT | IDENTITY_AND_ACCESS_CONTROL | 14.2 | |
| DEFAULT_PASSWORD_OF_SAP_STAR | IDENTITY_AND_ACCESS_CONTROL | 14.3 | |
| EXISTENCE_OF_SAP_STAR | IDENTITY_AND_ACCESS_CONTROL | 14.3 | valid from 2026-06-16 |
| LOCK_STATUS_OF_SAP_STAR | IDENTITY_AND_ACCESS_CONTROL | 14.3 | valid from 2026-06-16 |

---

## Example Calculation (SYSTEM STATUS)

Suppose `NO_OF_DUMPS` has a current raw value of 200 failures/day:
- x₀ = 400, k = −0.05
- compliance = (1 / (1 + exp(−(−0.05) × (200 − 400)))) × 100
- = (1 / (1 + exp(−10))) × 100
- ≈ 99.995% → **GREEN**

At the SLA point (400 dumps): compliance = 50%.
At 800 dumps (double the SLA): compliance ≈ 0.007% → **RED**.

---

## AI-Assisted Operations Score Optimization — Metric Priority List

The **Metric Priority List** answers the question: *"Which metrics should an operator focus on first to improve the Operations Score as much as possible?"*

### End-to-end contribution weight

Because the Operations Score is a three-level weighted product (metric → KPI → Area Score → Operations Score), every metric has a scalar **end-to-end weight** c_i that captures its total leverage on the final score:

```
c_i = (metric_weight_i / 100) × (kpi_weight_j / 100) × area_score_weight_k
```

Where:
- `metric_weight_i` — weight of metric i within its KPI
- `kpi_weight_j` — weight of KPI j within its Area Score
- `area_score_weight_k` — weight of the Area Score in the Operations Score (System Health 0.3, Operations Setup 0.3, Security 0.4)

This is a constant per metric — it does not depend on the current metric value. It represents the maximum possible contribution of that metric to the Operations Score (achieved when compliance = 100%).

### Two priority views

The Priority List offers two ranked views:

#### 1. Current Deficit

Shows metrics that are **currently dragging the score down**, sorted worst-first.

```
deficit_i = c_i × (compliance_i / 100 − 1)    [range: −c_i .. 0]
```

Always ≤ 0. A fully-compliant metric contributes 0 deficit; a metric at 0% compliance contributes −c_i (the entire weight is lost). Rank 1 is the metric causing the largest absolute gap between the current score and a perfect score.

#### 2. Potential Gain

Shows metrics where **fixing them to 100% compliance would yield the largest score improvement**, sorted best-first.

```
gain_i = c_i × (1 − compliance_i / 100)    [range: 0 .. c_i]
```

Always ≥ 0. A fully-compliant metric has zero gain; a metric at 0% has gain = c_i (full weight is recoverable). Rank 1 is the metric where remediation delivers the maximum improvement to the Operations Score.

Note that Current Deficit and Potential Gain are symmetric: `deficit_i = −gain_i`. They differ only in how they frame the same quantity — one emphasizes the current damage, the other the potential upside.

### Example

ABAP.System.Heartbeat at 50% compliance:
- metric_weight = 90% within AVAILABILITY KPI
- kpi_weight = 50% within System Health Area Score
- area_score_weight = 30% (System Health)

```
c_i    = 0.9 × 0.5 × 0.3          = 0.135
gain   = 0.135 × (1 − 0.50)       = +6.75%  potential gain on the Operations Score
deficit = 0.135 × (0.50 − 1)      = −6.75%  current drag on the Operations Score
```

This matches the "+6.75%" shown for ABAP.System.Heartbeat at rank 1 in the Potential Gain view in the dashboard.

---

## Execute — Automatable vs. Remediation-Plan Metrics

The agent's Execute rung acts on the Metric Priority List. Not every metric can be
fixed by the agent: the deciding factor is **whether the fix is a Cloud ALM
configuration action the agent can perform, or a change to the managed system that
only a human (Basis/Security) can apply.** Every metric falls into exactly one of
two buckets.

### Bucket A — Automatable (agent activates the capability directly)

These metrics score 0 because a monitoring capability or security check is
**inactive / not configured**, not because the underlying system is unhealthy.
Activating them is a configuration action inside Cloud ALM (a toggle / enable), it
does not touch the managed system, and it is reversible. The agent may perform these
after explicit user confirmation.

- **Operations Setup — all monitoring-setup KPIs.** Every binary
  `CONFIGURATION_STATUS` / `RUNTIME_STATUS` metric where the capability is inactive:
  - Health Monitoring setup
  - Exception Monitoring setup — **including the individual categories** (runtime
    errors, system logs, application logs) that can be activated independently
  - Integration Monitoring setup
  - Job & Automation Monitoring setup
  - Configuration & Security Analysis setup
  - Real User Monitoring setup
  - Business Service Management setup
  - Business Process Monitoring setup
- **Security Score — inactive CSA compliance checks.** A `SYSTEM_HARDENING`,
  `VULNERABILITY_AND_LIFECYCLE`, or `IDENTITY_AND_ACCESS_CONTROL` check that scores 0
  because the check itself is switched off (not being evaluated). The agent enables
  the check.

### Bucket B — Remediation-plan only (agent guides, cannot execute)

These metrics reflect the **actual runtime behaviour or configuration of the managed
system**. The agent cannot change them by configuration; it produces a structured,
metric-specific remediation plan and hands off to the responsible human.

- **System Health — all metrics.** These measure how the system is actually
  behaving, so they cannot be "activated" to compliance:
  - ABAP / instance heartbeats (availability)
  - Number of dumps, syslog errors, application-log errors (exceptions)
  - Failed jobs (background)
  - **Average / maximum ABAP response time (performance)** — the canonical
    non-configurable example; a slow system is not fixed by a toggle.
- **Security Score — genuinely failing checks.** A check that is active and
  evaluating but returns NONCOMPLIANT because the underlying setting is wrong
  (e.g. overdue support packages, aged kernel, a misconfigured hardening parameter).
  Basis/Security must apply the change in the system.

### The activation ≠ compliance rule

Activating a Bucket A capability makes it **start evaluating** — it does not
guarantee a score gain. Two outcomes are possible once the first evaluation runs:

1. **Compliant** — the underlying setting was already correct; the 0 was purely a
   monitoring blind spot. The full potential gain (`c_i`) is realised.
2. **Noncompliant** — activation reveals a real finding. The metric stays at 0, and
   the item **moves from Bucket A to Bucket B**: the agent now produces a remediation
   plan for it instead of claiming a gain.

The agent must state this honestly before activating: enabling restores *visibility*,
and the projected gain is only realised if the capability comes back compliant.

### Execution rules for the agent

- Never execute any activation without explicit user confirmation (hard boundary).
- Batch activations are allowed but must be confirmed as a set, with per-capability
  progress and a per-capability outcome (compliant vs. finding).
- Any capability that returns a finding after activation is reported and converted to
  a remediation-plan item — never left as a silent partial success.
- Bucket B metrics are never presented as automatable; the agent offers a
  step-by-step remediation plan and, where relevant, the responsible role
  (Basis / Security administrator).
