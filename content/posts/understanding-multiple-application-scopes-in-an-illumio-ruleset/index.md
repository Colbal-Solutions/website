---
title: Understanding Multiple Application Scopes in an Illumio Ruleset
featured_image: illumio-logo.webp
author: Charles Panter
date: 2026-04-08T15:32:52.801Z
description: "Illumio Core ruleset scope behavior explained: multiple
  application labels are evaluated in isolation, not as a union. Discover best
  practices for intra-scope rules, extra-scope rules, and cross-application
  policy configuration."
tags:
  - 9:27 AMruleset
  - scope
  - intra-scope rules
  - extra-scope rules
  - application label
  - policy configuration
  - PCE
  - cross-application communication
  - label expansion
  - Illumio
---


**Summary**

The Illumio Core UI allows users to add more than one scope (including multiple Application labels) to a single ruleset. While this is valid configuration, the behavior is frequently misunderstood. Multiple scopes in a ruleset are evaluated **in isolation** — they do not create a union or allow cross-communication between the scoped groups.

Administrators sometimes add multiple application labels to a ruleset scope expecting that intra-scope rules will permit communication **between** those applications. In practice, this does not occur. The rules are applied to each scope independently.

**How It Works**

**Scope Evaluation Is Per-Scope, Not Combined**

When a ruleset contains multiple scopes, the PCE treats each scope as a separate boundary. Any intra-scope rules defined in the ruleset are duplicated and applied within each scope individually.

**Example:**

A ruleset is created with two scopes:

| S﻿cope | A﻿pp | E﻿nv  | L﻿oc |
| ------ | ---- | ----- | ---- |
| 1﻿     | H﻿RM | P﻿rod | U﻿S  |
| 2﻿     | H﻿RM | D﻿ev  | U﻿S  |

An intra-scope rule is added: `DB ← App (All Services)`

**What the PCE computes:**

* HRM / Prod / US: DB can receive traffic from App ![✅](https://fonts.gstatic.com/s/e/notoemoji/17.0/2705/32.png)
* HRM / Dev / US: DB can receive traffic from App ![✅](https://fonts.gstatic.com/s/e/notoemoji/17.0/2705/32.png)
* HRM / Prod / US ↔ HRM / Dev / US: **No communication allowed** ![❌](https://fonts.gstatic.com/s/e/notoemoji/17.0/274c/32.png)

The rule is effectively expanded into two independent rules — one per scope.

**Label Expansion Behavior**

When the same label type appears multiple times within a rule, the PCE expands them into separate rules, each containing a single label per type. This expansion is the mechanism behind the per-scope isolation.

**Intra-Scope Label Matching**

For intra-scope rules, both the source and destination must match the labels defined in the scope. This is a hard constraint — the scope acts as a filter on both sides of the rule. Because each scope is evaluated independently, there is no path for traffic to flow between workloads that belong to different scopes.

- - -

**Common Misconception**

| W﻿hat users expect                                                                                                    | W﻿hat actually happens                                                                                     |
| --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| A﻿dding app "HRM" and app "ERP" to a ruleset scope allows HRM and ERP workloads to communicate via intra-scope rules. | I﻿ntra-scope rules are applied **within** HRM and **within** ERP separately. No HRM <-> ERP commumication. |

- - -

**When You Need Cross-Application Communication**

If the goal is to allow traffic **between** two applications, use one of the following approaches:

**Option 1: Extra-Scope Rules (not recommended)**

Extra-scope rules allow communication from sources inside the scope to destinations outside the scope (or vice versa). The source side is bound to the scope, but the destination side is "global" and can reference labels from a different application.

**Example:** To allow HRM's web tier to reach ERP's API tier, create an extra-scope rule in the HRM ruleset where the global destination specifies the ERP application label and the appropriate role label.

**Option 2: Separate Rulesets with Explicit Rules (recommended)**

Create a dedicated ruleset for each application and use extra-scope rules to reference the other application explicitly. This approach is often clearer and easier to audit.

**Option 3: Broad Scope with Specific Rules (infrastructure)**

For shared infrastructure services (e.g., DNS, Active Directory, backup), a ruleset scoped to All Applications / All Environments / All Locations may be appropriate. Use caution with this approach — an overly permissive intra-scope rule in a broad scope can inadvertently allow all-to-all communication across the entire environment.

- - -

**Best Practices**

1. **Use one Application label per scope when possible.** This makes policy intent clear and avoids confusion about cross-app behavior.
2. **Be cautious combining multiple label types.** Adding multiple Application labels alongside an Environment label creates combinatorial expansion that can produce unintended results.
3. **Use extra-scope rules for inter-application traffic.** This is the intended mechanism for cross-boundary communication and makes the policy explicitly visible.
4. **U﻿se label groups**. If applications (or any label type) need to be treated as one entity, a label group can be created to accomplish this. 
5. **Use Traffic to validate.** After provisioning, use the Traffic feature (Explore > Traffic) to verify which rules are actually applied to specific workloads. This helps catch overly broad or missing rules. 
6. **Limit rules per ruleset.** Illumio recommends no more than 500 rules per ruleset for UI performance. Split large rule sets across multiple rulesets or use the REST API if needed.