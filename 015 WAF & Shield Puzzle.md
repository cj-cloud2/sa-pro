### WAF \& Shield Puzzle: "Attack Avalanche – Secure the Flood"

**Puzzle Scenario**
You're architecting SecureShop, a web app behind ALB. Normal traffic: 10k RPM users from global IPs. Sudden surge: 100k RPM with bots/SQLi patterns from 5 ASNs. WAF blocks 60% (false positives on legit EU traffic), Shield alerts but no mitigation. App latency spikes to 5s, 20% 5xx errors.

**Provided Clues:**

```
WAF Rules (priority order):
1. Rate-based: 2k RPM/5min per IP → BLOCK
2. AWSManagedRulesCommonRuleSet (SQLi, XSS)
3. Custom: Regex /evilbot/ → COUNT
4. Geo: Block Asia → ALLOW (override)

Shield: Standard tier, ALB association.
CloudWatch: 80% rule matches on rate-limit, DDoS alarms firing.
Traffic: Layer 7 HTTP floods, no SYN floods.
```

**Your Task (10-15 mins):** Answer these 3 questions conceptually. Use the clues above as guiding hints.

1. **Why does Shield Standard fail to mitigate this attack?**
2. **What's causing false positives blocking legitimate traffic, and how does rule order contribute?**
3. **How should you conceptually prioritize WAF rule groups to balance protection vs. availability?**

**Guiding Hints (Embedded):**

- Shield clue: Volumetric Layer 7 vs. network floods? Tiers differ how?
- False positive hint: Rate-limit hits 80%, geo override—evaluation flow?
- Prioritization hint: Managed vs. custom—chaining and actions (BLOCK/COUNT)?

***

### Detailed Solutions with Teaching Justifications

#### 1. Shield Standard Fails to Mitigate the Attack

**Answer:** Shield Standard auto-protects against common Layer 3/4 DDoS (e.g., SYN floods) at no cost, but this Layer 7 HTTP flood requires Shield Advanced for intelligent mitigation like TLS termination and origin shielding.

**Teaching Justification from Scratch:**
AWS Shield is DDoS protection; think of it as a global shield around your resources.

- **Basics:** DDoS attacks overwhelm capacity. Layer 3/4 = network (UDP floods, SYN). Layer 7 = app (HTTP GET floods).
- **Shield Tiers:**


| Tier | Coverage | Key Features | Cost |
| :-- | :-- | :-- | :-- |
| Standard | All AWS customers, auto | Layer 3/4 basics, always-on | Free |
| Advanced | Opt-in subscription | Layer 7, WAF integration, 24/7 concierge | \$3k/mo + usage |

Standard absorbs ~5-10 Tbps via AWS edge, but passes Layer 7 to your ALB (clue: no SYN floods). Result: ALB overwhelmed, latency spikes.
- **EKS/ALB Tie-In:** ALB sees floods as "legit" traffic → queues build → 5xx.
- **Conceptual Fix:** Upgrade to Advanced: Proactive engagement, Global Accelerator integration for anycast routing. Justification: Layer 7 needs app-aware scrubbing (e.g., JS challenges)—Standard is reactive network-only, Advanced is proactive (99.99%+ efficacy per AWS benchmarks).


#### 2. False Positives from Rule Order

**Answer:** Rate-based rule (priority 1) triggers first on bursty legit traffic (e.g., EU flash sale), blocking before geo/custom rules evaluate. 80% matches + geo ALLOW override ignored due to BLOCK action terminating flow.

**Teaching Justification from Scratch:**
WAF evaluates rules sequentially (priority 1-N); first match dictates action (ALLOW/BLOCK/COUNT/CAPTCHA).

- **Basics:** Rules inspect headers/body/query. Actions:


| Action | Effect |
| :-- | :-- |
| BLOCK | Drop + 403 |
| COUNT | Log, continue |
| CAPTCHA | Challenge bots |

- **Evaluation Flow:** Top-down, no fallthrough on terminal actions. Here:

1. Rate-limit (2k/5min/IP): Legit users burst → BLOCK (ends eval).
2. Skips SQLi/XSS (would catch bots).
3. Geo ALLOW never reached.
- **Clue Tie-In:** 100k RPM floods mimic legit bursts; Asia geo irrelevant for EU false positives.
- **Conceptual Fix:** Move rate-based lower, use IP sets for known-good ASNs. Justification: Prioritizes precision—early BLOCKs amplify false positives (WAF best practice: COUNT for tuning, then BLOCK).


#### 3. Prioritizing WAF Rule Groups

**Answer:** Group high-confidence managed rules (SQLi/XSS) first (COUNT), then rate-based/custom (ALLOW overrides), ending with geo. Use actions chaining: COUNT → inspect → targeted BLOCK.

**Teaching Justification from Scratch:**
Rule groups bundle related rules (e.g., AWSManagedRulesSQLiRuleSet). Associate to WebACL with priority.

- **Basics:** WebACL = ordered list of rules/groups. Evaluation: Group internally (top-down), then next group.

```mermaid
Traffic → RuleGrp1 (pri 0) → RuleGrp2 (pri 1) → Default Action
```

- **Best Practice Prioritization:**

1. **Signature-based (pri 0):** CoreRuleSet (OWASP Top 10) → COUNT (low false +).
2. **Behavioral (pri 1):** Rate-based, Bot Control → CAPTCHA.
3. **Custom/Contextual (pri 2):** Regex/geo → BLOCK.
4. Default: ALLOW.
- **Clue Fix:** Reorder: CommonRuleSet (catches SQLi pre-rate), rate-based COUNT (observe), geo last.
- **Shield Synergy:** Advanced Shield feeds WAF with threat intel (auto-rule updates). Justification: Reduces eval overhead (fewer matches), boosts accuracy—e.g., 90%+ threat block rate per AWS case studies, embodying "defense in depth" (prevent > detect > respond).

This puzzle teaches WAF/Shield from layers (network→app) to tuning (order→actions), building cloud-native security intuition.

