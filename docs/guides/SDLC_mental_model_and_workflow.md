# Mental model of Agentic-driven Software Development

## 1. Mental models
### Question: 
Do Engineers Review All AI-Generated Code? 

### Answer:
No, Not Realistically. Engineers cannot meaningfully review every line at the same depth they would review human-written code. The volume is too high. Review speed does not scale with generation speed.

---
### Three Tiers of Review in Reality

Rather than pretending all code gets equal review, teams should adopt an explicit tiered strategy:

| Tier                     | Code Type                                                                 | Review Approach                                                                 |
|--------------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| **Tier 1 — Deep**        | Auth, payments, core business logic, data models, security boundaries    | Line-by-line audit, adversarial testing, invariant verification                 |
| **Tier 2 — Structural**  | Service logic, API handlers, DB queries                                  | Review structure, data flow, edge cases; spot-check implementation              |
| **Tier 3 — Trust with Gates** | Boilerplate, glue code, CRUD, UI scaffolding                        | Review only that tests pass and contracts are satisfied; skip line-by-line      |

> The critical discipline is knowing which tier each piece belongs to — and never accidentally treating Tier 1 code as Tier 3.

**Core Priciple**

> The mental model shifts from "I understand this code" to "I understand what this code must satisfy, and I can verify it does." The engineers who navigate this well are the ones who invest in specs, invariants, tests, and observability — and treat code review as an audit against those artifacts, not a reading exercise.

---

### Question: 

Handling Production Incidents When Nobody Deeply Knows all the Code

### Answer:

Since engineers do have deep knowledge of Tier 1 and meaningful structural knowledge of Tier 2, the incident response approach can be more targeted:


1. Triage starts with tier ownership: First question during an incident: "Which tier is the failing component?"

- Tier 1 failure -> assign the engineer who reviewed that code. They lead diagnosis directly — observability confirms what they already suspect.
- Tier 2 failure -> engineer uses structural knowledge to narrow blast radius, then traces logs to confirm root cause.
- Tier 3 failure -> go straight to observability + AI-assisted diagnosis. No engineer is expected to know this deeply.

2. Domain-level alerts, not just technical metrics. Alert on business invariants, not just CPU/latency (Optional):

- "Payment charged but no fulfillment record created"
- "Login success rate dropped below 95%"

    These tell you what is broken without reading code — critical for Tier 3 failures.

3. Structured logs with end-to-end request tracing
Every request must carry a correlation ID through the entire system. Trace one bad request through logs — no code reading required.

4. AI-assisted fix — but gated strictly During incident:

- Feed AI the relevant logs + error + system map context
- AI proposes hypothesis and fix
- Human must approve before anything touches production

Never let AI push a fix unsupervised during an incident.

**Bottom line**
> Tier 1 and Tier 2 ownership means critical path incidents have a human expert who can reason directly. For Tier 3 — where no engineer has deep knowledge — domain-level alerts, distributed tracing, and AI-assisted diagnosis from live logs are the safety net. The system must be able to tell you what is wrong, because no human can.

---
---

2. Practical Workflows                                                    
                                                                            
2.0 Prerequisite — Code Review Tier Model                               
                                                                        
Before any workflow runs, the team agrees on a tier classification applied
to modules (not individual lines). Reviewed quarterly.                   
                                                                        
- Tier 1 — Critical. Core business logic, security boundaries, data       
models, payment/compliance/safety paths. Deep human review on every
change. AI proposes; human disposes.                                      
- Tier 2 — Standard. Shared infrastructure, reusable components, internal
APIs. Spot-check review. AI can ship with light human review.             
- Tier 3 — Ephemeral. Glue code, integrations, UI surface, internal tools,
one-off scripts. Regenerable. Review is for SLO compliance, not          
code-level scrutiny.                                                    
                                                                        
▎ A change that spans tiers inherits the highest tier it touches.         

---       

2.1 Greenfield — Build from scratch                                     
                                                                        
Intro: Build an application from scratch (Monolith, Decoupled,
Microservices, etc.). Pick one of two workflows based on the product's    
strategic role.        
                                                
                                                                        
2.1.1 Human-guided Greenfield                                           

- When to use: Long-term, scalable product expected to be maintained for  
years
- Who to use: Senior Developer, Solution Architect, Tech Lead             
                                                                        
Top-level workflow

[Human]         Prepare detailed PRD + UI design (optional)
    ↓                                                      
[Human | AI]    Break PRD into sub-PRDs (per domain/feature)              
    ↓                                                       
[Human + AI]    Code generation & validation loop (per sub-PRD)           
    ↓                                                                   
[Human]         Final E2E test + deploy                                   


Inner loop — Code generation & validation (per sub-PRD)                   
[AI]     Clarify requirements (ask questions)
    ↓                                                                     
[Human]  Answer clarifying questions                                    
    ↓                                                                     
[AI]     Propose solution & architecture (options + tradeoffs)
    ↓                                                                     
[Human]  Adjust solution & architecture in detail                       
    ↓                                            
[Human]  Classify modules into Tier 1 / 2 / 3                             
    ↓                                        
[AI]     Generate code + write tests + self-validate                      
    ↓                                                                   
[Human]  Review by tier (deep T1, spot T2, SLO-check T3)                  
    ↓                                                                   
[AI]     Address review feedback                                          
    ↓                           
(loop until accepted)                                                     
                                                                        
Human sign-off checkpoints: sub-PRD breakdown · architecture · tier
classification · per-tier review outcome · deploy.                        

---                               

2.1.2 Autonomous Greenfield                                             
                                                                        
- When to use: Market-fit testing, quick demo/proposal, feature-focused
(not system-focused)                                                      
- Who to use: Developer, Business User, PM/Designer                     
                                                                        
Top-level workflow
[Human]         Prepare PRD + UI design (optional)                        
    ↓                                                                   
[Human | AI]    Break PRD into sub-PRDs
    ↓                                  
[AI]            Code generation & validation loop (auto-approve)          
    ↓                                                           
[Human]         Final E2E test + deploy                                   
                                                                        
Inner loop — Auto-approve                                                 
[AI]     Clarify requirements
    ↓                                                                     
[Human]  Answer clarifying questions                                    
    ↓                               
[AI]     Propose + auto-approve solution & architecture
    ↓                                                  
[AI]     Generate code + self-validate                                    
    ↓                                 
[Human]  Final E2E test + deploy                                          

                                  
Guardrails (must be enforced to prevent misuse):                          
- Scope cap — single feature or demo app only
- Budget cap — token / wall-clock limits on agent runs                    
- Deploy target — staging/preview only; never prod without reclassifying
as Human-guided                                                           
- Graduation rule — if the product passes market-fit, it moves to 2.1.1   
before the first real user
                                                                        
---                    

2.2 Brownfield — Extend & maintain existing code                          
                                                                        
Intro: Develop and maintain an existing application (may include legacy
systems with thin/missing specs).                                         
                                                                        
Preconditions (one-time):                                                 
- [Human] Classify existing modules into Tier 1 / 2 / 3
- [Human] Document invariants for T1 modules (even one page each)         
- [Human + AI] Ensure verification stack exists (tests, types, lints, CI,
observability)                                                            
                                                                        
2.2.1 New feature / Bug fix                                               
                                                                        
[Human]  Input requirement (e.g., "add search filter to product listing")
    ↓
[AI]     Clarify requirements + locate affected modules                   
    ↓
[Human]  Answer clarifying questions                                      
    ↓                                                                   
[AI]     Propose solution & architecture (respect existing patterns)
    ↓                                                                     
[Human]  Adjust solution & architecture in detail
    ↓                                                                     
[Human]  Confirm tier of touched modules (highest wins)                 
    ↓                                                                     
[AI]     Generate code + tests + self-validate
    ↓                                                                     
[Human]  Review by tier                                                 
    ↓
[AI]     Address feedback
    ↓
[Human]  Final E2E test + deploy
                                                                        
2.2.2 Refactor / Migration (suggested addition)
                                                                        
When work is structural rather than feature-level:                        
[Human]  Define end-state architecture + invariants to preserve
    ↓                                                                     
[AI]     Generate migration plan (stepwise, each step reversible)       
    ↓                                                            
[Human]  Approve migration plan                                           
    ↓                          
[AI]     Execute step-by-step; commit per step; run full test suite       
    ↓                                                                   
[Human]  Review each step (T1 deep, T2/T3 sampled)
    ↓                                             
[Human]  Deploy progressively                                             
                            

2.2.3 Incident response (suggested addition)                              
                                                                        
When production is on fire:
[Human]       Triage symptom + identify affected module                   
    ↓                                                                   
[Human + AI]  Inspect traces/logs; AI proposes hypotheses
    ↓                                                    
[Human]       Decide: mitigate (rollback / flag) vs fix
    ↓                                                                     
[AI]          Generate patch + regression test
    ↓                                                                     
[Human]       Review (incident patches are ALWAYS T1, regardless of module tier)                                                                    
    ↓                                                                     
[Human]       Deploy with elevated monitoring
    ↓                                                                     
[Human]       Post-incident: update invariants, tests, tier if needed   

---                 
                                                    
Summary matrix
                                                                        
┌────────────────────┬────────┬────────────┬─────────┬───────────────┐  
│      Workflow      │ Speed  │   Risk     │ Human   │   Typical     │    
│                    │        │ tolerance  │  load   │    driver     │
├────────────────────┼────────┼────────────┼─────────┼───────────────┤    
│ 2.1.1 Human-guided │ Medium │ Low        │ High    │ Senior dev /  │  
│  Greenfield        │        │            │         │ architect     │
├────────────────────┼────────┼────────────┼─────────┼───────────────┤
│ 2.1.2 Autonomous   │ High   │ High       │ Low     │ PM / business │
│ Greenfield         │        │ (scoped)   │         │  user / dev   │    
├────────────────────┼────────┼────────────┼─────────┼───────────────┤
│ 2.2.1 Brownfield   │ Medium │ Varies by  │ Medium  │ Any dev       │    
│ feature/bugfix     │        │ tier       │         │               │    
├────────────────────┼────────┼────────────┼─────────┼───────────────┤
│ 2.2.2 Brownfield   │ Slow   │ Low        │ High    │ Senior dev    │    
│ refactor           │        │            │         │               │  
├────────────────────┼────────┼────────────┼─────────┼───────────────┤
│ 2.2.3 Incident     │ Fast   │ Low        │ High    │ Oncall        │
│ response           │        │            │         │               │    
└────────────────────┴────────┴────────────┴─────────┴───────────────┘