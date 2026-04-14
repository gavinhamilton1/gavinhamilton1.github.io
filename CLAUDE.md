# CIB Unified Ingress — Claude Code Context

## Overview

This is the working context for the CIB Unified Ingress platform project. This document is maintained for use with Claude Code and should be treated as the authoritative context for all AI-assisted work on this project.

**Owner:** Gavin Hamilton, Head of Architecture, JPMorgan Chase CIB Digital Enablement  
**Executive Sponsor:** Sri (Payments CIO)  
**Status:** Architecture phase — plan under revision following strategic direction change  
**Delivery target:** End of year (Dec 31, 2026)

---

## Strategic Direction (Updated)

Following a review with Sri (Payments CIO), the platform scope has been elevated from a well-engineered ingress platform to a next-generation enterprise ingress capability. Two key additions:

### 1. Active/Active CDN (Akamai + Cloudflare)
- Both Akamai and Cloudflare are **primary** — no primary/standby model
- WAF-as-code: single GitOps source of truth for all WAF rules, OWASP policies, custom CIB rules, bot management, and TLS config. CI/CD pipeline pushes to both vendor APIs on every merge
- Both CDN networks active simultaneously via Anycast — clients resolve to closest/fastest PoP across either provider
- Eliminates WAF consistency risk through engineering rather than accepting it as a constraint
- Rationale: at JPM scale a 30-minute outage costs hundreds of millions. The cost/complexity argument for primary/standby does not hold

### 2. VAN / Private Circuit Ingress
- JPM has existing leased line support for certain institutional clients
- Platform must accommodate private network ingress alongside public internet
- VAN clients bypass DNS resolution, L1 GTM, and L2 CDN/WAF entirely
- VAN clients enter at L3 via PSaaS+ Private Circuit (on-prem path) or AWS Direct Connect
- Auth model for VAN: mTLS certificate binding in place of DPoP
- Same OPA policy engine, separate VAN origin policy branch
- Same gateway infrastructure (Envoy/Kong) — convergence at L4
- Zero trust model: VAN clients are identity-verified, not network-trusted

### 3. End of Year Delivery
- Current plan: 48 weeks (Apr 28 → Feb 2027)
- Required: 35 weeks (Apr 28 → Dec 31, 2026)
- Gap: 13 weeks — requires more parallel execution and additional resourcing
- Resourcing is flexible (Gavin has headcount authority)
- LOB adoption not in scope for Dec 31 — platform functional, adoption follows Q1 2027
- Plan rebuild required around the new architecture

---

## Architecture — Dual Ingress Path Model

Two distinct ingress paths converge at L4:

### Public Internet Path
```
DNS (Self-hosted NS + Cloudflare NS, active/active)
  ↓
L1: Akamai GTM + Cloudflare LB (both active, config-synced)
  ↓
L2: Akamai Ion/Kona WAF + Cloudflare Edge WAF (both active, WAF-as-code synced)
  ↓
L3: PSaaS+ (GKP path) / CTC Edge (AWS path)
  ↓
L4: Envoy Web Gateway / Kong API Gateway [CONVERGENCE]
  ↓
L5: On-Prem (GAP, GKP, VSI) + AWS (EKS, ECS, EC2, Lambda)
```

### VAN / Private Circuit Path
```
Leased line (no DNS, no CDN/WAF)
  ↓
L3: PSaaS+ Private Circuit (on-prem) / AWS Direct Connect
  ↓
L4: Envoy Web Gateway / Kong API Gateway [CONVERGENCE]
  ↓
L5: On-Prem (GAP, GKP, VSI) + AWS (EKS, ECS, EC2, Lambda)
```

### L4 Auth Pipeline (per origin type)
- **Internet clients:** jwt_authn → Session Validator (DPoP) → OPA PDP → Context Propagator
- **VAN clients:** mTLS cert → Session Validator (mTLS) → OPA PDP (VAN policy branch) → Context Propagator

### Key Architectural Decisions
- **Token model:** Session JWT carries identity only (sub, sid, entity, client type, DPoP thumbprint). No entitlement claims. Entitlements resolved dynamically from ABAC attribute source at request time via OPA PDP (PIP pattern)
- **OPA is the PDP:** Envoy ext_authz is the PEP (Policy Enforcement Point). OPA is the PDP (Policy Decision Point)
- **SpiceDB at L5 only:** Opt-in ReBAC pattern for service teams. Not in the L4 request path
- **WAF-as-code:** Single source of truth for all WAF rules across both CDN vendors. Lives in Ingress Control Plane
- **Regional failover:** Automatic below L2. GTM composite health check triggers DNS steering. Session Manager active/active ensures zero session loss on failover
- **Sidecar isolation:** OPA PDP, Session Validator, and OTEL Collector run as sidecars per gateway pod. Failure is pod-scoped, not platform-wide

---

## Architecture Diagram SVG

The following SVG is the current architecture overview diagram embedded in the project plan HTML. It shows the dual-path model (public internet and VAN/private circuit).

```html
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1060 640" width="100%" style="display:block;font-family:Arial,sans-serif">
    <defs>
      <marker id="arrD" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,1 L6,3.5 L0,6 Z" fill="#c8d8f0"/></marker>
      <marker id="arrP" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,1 L6,3.5 L0,6 Z" fill="#7a72c8"/></marker>
      <marker id="arrT" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,1 L6,3.5 L0,6 Z" fill="#1a6a5a"/></marker>
      <pattern id="bypass" patternUnits="userSpaceOnUse" width="10" height="10">
        <line x1="0" y1="10" x2="10" y2="0" stroke="#d4dde8" stroke-width="1"/>
      </pattern>
    </defs>

    <!-- BACKGROUND -->
    <rect width="1060" height="640" fill="#f8fafd" rx="8" stroke="#d0dae8" stroke-width="1"/>

    <!-- TITLE BAR -->
    <rect x="0" y="0" width="1060" height="32" fill="#1a3a6a" rx="8"/>
    <rect x="0" y="20" width="1060" height="12" fill="#1a3a6a"/>
    <text x="530" y="21" text-anchor="middle" font-size="13" font-weight="700" fill="white">CIB Unified Ingress — Architecture Overview</text>

    <!-- COLUMN HEADERS -->
    <rect x="90" y="32" width="592" height="24" fill="#1e4d8c"/>
    <text x="386" y="48" text-anchor="middle" font-size="10" font-weight="700" fill="white" letter-spacing="0.1em">PUBLIC INTERNET PATH</text>
    <rect x="688" y="32" width="372" height="24" fill="#1a5a4a"/>
    <text x="874" y="48" text-anchor="middle" font-size="10" font-weight="700" fill="white" letter-spacing="0.1em">VAN / PRIVATE CIRCUIT PATH</text>

    <!-- VERTICAL DIVIDER (DNS through L3 only) -->
    <line x1="684" y1="56" x2="684" y2="357" stroke="#c8d8e8" stroke-width="2"/>

    <!-- ═══ DNS LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="56" width="1060" height="70" fill="#f2faf5"/>
    <rect x="0" y="56" width="90" height="70" fill="#2a6a4a"/>
    <text x="45" y="85" text-anchor="middle" font-size="14" font-weight="700" fill="white">DNS</text>
    <text x="45" y="100" text-anchor="middle" font-size="7.5" fill="#a8d8b8" letter-spacing="0.08em">NAMESERVERS</text>
    <text x="45" y="114" text-anchor="middle" font-size="7" fill="#a8d8b8" letter-spacing="0.06em">ACTIVE / ACTIVE</text>

    <!-- Self-hosted NS -->
    <rect x="98" y="63" width="268" height="56" fill="white" rx="5" stroke="#b0d0c0" stroke-width="1"/>
    <rect x="98" y="63" width="268" height="18" fill="#2a6a4a" rx="5"/>
    <rect x="98" y="71" width="268" height="10" fill="#2a6a4a"/>
    <text x="232" y="76" text-anchor="middle" font-size="11" font-weight="700" fill="white">Self-Hosted Nameservers</text>
    <text x="232" y="93" text-anchor="middle" font-size="10" fill="#5a7a9a">JPM-operated authoritative NS</text>
    <text x="232" y="107" text-anchor="middle" font-size="9.5" fill="#2a6a4a" font-weight="600">Active · Full organisational control</text>

    <!-- Zone sync indicator -->
    <text x="387" y="89" text-anchor="middle" font-size="16" fill="#2a6a4a">⇄</text>
    <text x="387" y="102" text-anchor="middle" font-size="7.5" fill="#2a6a4a">zone sync</text>

    <!-- Cloudflare NS -->
    <rect x="406" y="63" width="268" height="56" fill="white" rx="5" stroke="#b0d0c0" stroke-width="1"/>
    <rect x="406" y="63" width="268" height="18" fill="#2a6a4a" rx="5"/>
    <rect x="406" y="71" width="268" height="10" fill="#2a6a4a"/>
    <text x="540" y="76" text-anchor="middle" font-size="11" font-weight="700" fill="white">Cloudflare Nameservers</text>
    <text x="540" y="93" text-anchor="middle" font-size="10" fill="#5a7a9a">Cloud-operated authoritative NS</text>
    <text x="540" y="107" text-anchor="middle" font-size="9.5" fill="#2a6a4a" font-weight="600">Active · Global Anycast network</text>

    <!-- VAN DNS bypass -->
    <rect x="688" y="56" width="372" height="70" fill="url(#bypass)"/>
    <rect x="688" y="56" width="372" height="70" fill="rgba(240,246,250,0.88)"/>
    <text x="874" y="86" text-anchor="middle" font-size="11" font-weight="600" fill="#9aaaba">Not Applicable</text>
    <text x="874" y="102" text-anchor="middle" font-size="10" fill="#aabac8" font-style="italic">Dedicated leased line — no DNS resolution required</text>
    <text x="874" y="115" text-anchor="middle" font-size="10" fill="#aabac8" font-style="italic">VAN clients enter the platform directly at L3</text>
    <line x1="0" y1="126" x2="1060" y2="126" stroke="#d0dae8" stroke-width="0.5"/>

    <!-- ═══ L1 LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="126" width="1060" height="78" fill="#f0f4fc"/>
    <rect x="0" y="126" width="90" height="78" fill="#5a7aaa"/>
    <text x="45" y="159" text-anchor="middle" font-size="16" font-weight="700" fill="white">L1</text>
    <text x="45" y="175" text-anchor="middle" font-size="7.5" fill="#ccdcee" letter-spacing="0.08em">GLOBAL TRAFFIC</text>
    <text x="45" y="187" text-anchor="middle" font-size="7.5" fill="#ccdcee" letter-spacing="0.08em">MANAGEMENT</text>

    <!-- Akamai GTM -->
    <rect x="98" y="133" width="268" height="64" fill="white" rx="5" stroke="#c8d8f0" stroke-width="1"/>
    <rect x="98" y="133" width="268" height="18" fill="#5a7aaa" rx="5"/>
    <rect x="98" y="141" width="268" height="10" fill="#5a7aaa"/>
    <text x="232" y="146" text-anchor="middle" font-size="11" font-weight="700" fill="white">Akamai GTM</text>
    <text x="232" y="163" text-anchor="middle" font-size="10" fill="#5a7a9a">GeoDNS · Composite health checks</text>
    <text x="232" y="177" text-anchor="middle" font-size="10" fill="#5a7a9a">Regional failover · Hysteresis · TTL tuning</text>
    <text x="232" y="189" text-anchor="middle" font-size="9" fill="#5a7aaa" font-style="italic" font-weight="600">Active</text>

    <!-- Config sync indicator L1 -->
    <text x="387" y="163" text-anchor="middle" font-size="16" fill="#5a7aaa">⇄</text>
    <text x="387" y="174" text-anchor="middle" font-size="7.5" fill="#5a7aaa">config sync</text>

    <!-- Cloudflare LB -->
    <rect x="406" y="133" width="268" height="64" fill="white" rx="5" stroke="#c8d8f0" stroke-width="1"/>
    <rect x="406" y="133" width="268" height="18" fill="#5a7aaa" rx="5"/>
    <rect x="406" y="141" width="268" height="10" fill="#5a7aaa"/>
    <text x="540" y="146" text-anchor="middle" font-size="11" font-weight="700" fill="white">Cloudflare Load Balancer</text>
    <text x="540" y="163" text-anchor="middle" font-size="10" fill="#5a7a9a">GeoDNS · Health checks</text>
    <text x="540" y="177" text-anchor="middle" font-size="10" fill="#5a7a9a">Regional failover · Traffic weight control</text>
    <text x="540" y="189" text-anchor="middle" font-size="9" fill="#5a7aaa" font-style="italic" font-weight="600">Active</text>

    <!-- VAN L1 bypass -->
    <rect x="688" y="126" width="372" height="78" fill="url(#bypass)"/>
    <rect x="688" y="126" width="372" height="78" fill="rgba(240,246,250,0.88)"/>
    <text x="874" y="162" text-anchor="middle" font-size="11" font-weight="600" fill="#9aaaba">Not Applicable</text>
    <text x="874" y="178" text-anchor="middle" font-size="10" fill="#aabac8" font-style="italic">Private circuit bypasses traffic management</text>
    <text x="874" y="191" text-anchor="middle" font-size="10" fill="#aabac8" font-style="italic">No GTM involvement</text>
    <line x1="0" y1="204" x2="1060" y2="204" stroke="#d0dae8" stroke-width="0.5"/>

    <!-- ═══ L2 LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="204" width="1060" height="80" fill="#fdf3f0"/>
    <rect x="0" y="204" width="90" height="80" fill="#aa6060"/>
    <text x="45" y="237" text-anchor="middle" font-size="16" font-weight="700" fill="white">L2</text>
    <text x="45" y="253" text-anchor="middle" font-size="7.5" fill="#eeccc8" letter-spacing="0.08em">EDGE SECURITY</text>
    <text x="45" y="265" text-anchor="middle" font-size="7.5" fill="#eeccc8" letter-spacing="0.08em">CDN / WAF</text>

    <!-- Akamai Ion/Kona -->
    <rect x="98" y="211" width="268" height="66" fill="white" rx="5" stroke="#e8c0b8" stroke-width="1"/>
    <rect x="98" y="211" width="268" height="18" fill="#aa6060" rx="5"/>
    <rect x="98" y="219" width="268" height="10" fill="#aa6060"/>
    <text x="232" y="224" text-anchor="middle" font-size="11" font-weight="700" fill="white">Akamai Ion CDN + Kona WAF</text>
    <text x="232" y="241" text-anchor="middle" font-size="10" fill="#5a7a9a">Cache · Compress · DDoS · Bot mgmt</text>
    <text x="232" y="255" text-anchor="middle" font-size="10" fill="#5a7a9a">OWASP CRS · TLS termination · Edge certs</text>
    <text x="232" y="269" text-anchor="middle" font-size="9" fill="#aa6060" font-style="italic" font-weight="600">Active</text>

    <!-- WAF-as-code sync box -->
    <rect x="370" y="224" width="34" height="38" fill="#fff8f0" rx="4" stroke="#f0c870" stroke-width="1.5"/>
    <text x="387" y="237" text-anchor="middle" font-size="8" fill="#8a5200" font-weight="700">WAF</text>
    <text x="387" y="248" text-anchor="middle" font-size="8" fill="#8a5200" font-weight="700">sync</text>
    <text x="387" y="259" text-anchor="middle" font-size="7.5" fill="#8a5200">GitOps</text>

    <!-- Cloudflare Edge WAF -->
    <rect x="408" y="211" width="266" height="66" fill="white" rx="5" stroke="#e8c0b8" stroke-width="1"/>
    <rect x="408" y="211" width="266" height="18" fill="#aa6060" rx="5"/>
    <rect x="408" y="219" width="266" height="10" fill="#aa6060"/>
    <text x="541" y="224" text-anchor="middle" font-size="11" font-weight="700" fill="white">Cloudflare Edge + WAF</text>
    <text x="541" y="241" text-anchor="middle" font-size="10" fill="#5a7a9a">Cache · Compress · DDoS · Bot mgmt</text>
    <text x="541" y="255" text-anchor="middle" font-size="10" fill="#5a7a9a">OWASP CRS · TLS termination · Edge certs</text>
    <text x="541" y="269" text-anchor="middle" font-size="9" fill="#aa6060" font-style="italic" font-weight="600">Active</text>

    <!-- VAN L2 bypass -->
    <rect x="688" y="204" width="372" height="80" fill="url(#bypass)"/>
    <rect x="688" y="204" width="372" height="80" fill="rgba(240,246,250,0.88)"/>
    <text x="874" y="238" text-anchor="middle" font-size="11" font-weight="600" fill="#9aaaba">Not Applicable</text>
    <text x="874" y="254" text-anchor="middle" font-size="10" fill="#aabac8" font-style="italic">Private circuit bypasses CDN/WAF edge entirely</text>
    <text x="874" y="267" text-anchor="middle" font-size="9.5" fill="#aa6060" font-style="italic">mTLS at L4 compensates for absent edge WAF</text>
    <text x="874" y="280" text-anchor="middle" font-size="9" fill="#aabac8" font-style="italic">OPA VAN policy branch enforces additional controls</text>
    <line x1="0" y1="284" x2="1060" y2="284" stroke="#d0dae8" stroke-width="0.5"/>

    <!-- ═══ L3 LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="284" width="1060" height="73" fill="#f2f8f5"/>
    <rect x="0" y="284" width="90" height="73" fill="#4a8a62"/>
    <text x="45" y="317" text-anchor="middle" font-size="16" font-weight="700" fill="white">L3</text>
    <text x="45" y="332" text-anchor="middle" font-size="7.5" fill="#c0e0ce" letter-spacing="0.08em">NETWORK</text>
    <text x="45" y="344" text-anchor="middle" font-size="7.5" fill="#c0e0ce" letter-spacing="0.08em">PERIMETER</text>

    <!-- PSaaS+ GKP public -->
    <rect x="98" y="291" width="276" height="58" fill="white" rx="5" stroke="#b0d8c0" stroke-width="1"/>
    <rect x="98" y="291" width="276" height="18" fill="#4a8a62" rx="5"/>
    <rect x="98" y="299" width="276" height="10" fill="#4a8a62"/>
    <text x="236" y="304" text-anchor="middle" font-size="11" font-weight="700" fill="white">PSaaS+ (GKP path)</text>
    <text x="236" y="321" text-anchor="middle" font-size="10" fill="#5a7a9a">TLS re-origination · Perimeter routing</text>
    <text x="236" y="335" text-anchor="middle" font-size="9.5" fill="#4a8a62" font-weight="600">NA · EMEA · APAC</text>

    <!-- CTC Edge AWS public -->
    <rect x="384" y="291" width="290" height="58" fill="white" rx="5" stroke="#b0d8c0" stroke-width="1"/>
    <rect x="384" y="291" width="290" height="18" fill="#4a8a62" rx="5"/>
    <rect x="384" y="299" width="290" height="10" fill="#4a8a62"/>
    <text x="529" y="304" text-anchor="middle" font-size="11" font-weight="700" fill="white">CTC Edge (AWS path)</text>
    <text x="529" y="321" text-anchor="middle" font-size="10" fill="#5a7a9a">TLS re-origination · Perimeter routing</text>
    <text x="529" y="335" text-anchor="middle" font-size="9.5" fill="#4a8a62" font-weight="600">NA · EMEA · APAC</text>

    <!-- VAN entry header strip -->
    <rect x="688" y="284" width="372" height="14" fill="#1a5a4a"/>
    <text x="874" y="294" text-anchor="middle" font-size="8" font-weight="700" fill="white" letter-spacing="0.1em">VAN ENTRY POINT</text>

    <!-- PSaaS+ Private Circuit -->
    <rect x="694" y="299" width="174" height="52" fill="white" rx="5" stroke="#1a6a5a" stroke-width="1.5"/>
    <rect x="694" y="299" width="174" height="16" fill="#1a6a5a" rx="5"/>
    <rect x="694" y="307" width="174" height="8" fill="#1a6a5a"/>
    <text x="781" y="312" text-anchor="middle" font-size="10" font-weight="700" fill="white">PSaaS+ Private Circuit</text>
    <text x="781" y="327" text-anchor="middle" font-size="9.5" fill="#5a7a9a">Leased line termination</text>
    <text x="781" y="339" text-anchor="middle" font-size="9.5" fill="#5a7a9a">TLS re-origination</text>
    <text x="781" y="349" text-anchor="middle" font-size="8.5" fill="#1a6a5a" font-weight="600">On-premises path</text>

    <!-- AWS Direct Connect -->
    <rect x="876" y="299" width="176" height="52" fill="white" rx="5" stroke="#1a6a5a" stroke-width="1.5"/>
    <rect x="876" y="299" width="176" height="16" fill="#1a6a5a" rx="5"/>
    <rect x="876" y="307" width="176" height="8" fill="#1a6a5a"/>
    <text x="964" y="312" text-anchor="middle" font-size="10" font-weight="700" fill="white">AWS Direct Connect</text>
    <text x="964" y="327" text-anchor="middle" font-size="9.5" fill="#5a7a9a">Private cloud peering</text>
    <text x="964" y="339" text-anchor="middle" font-size="9.5" fill="#5a7a9a">TLS re-origination</text>
    <text x="964" y="349" text-anchor="middle" font-size="8.5" fill="#1a6a5a" font-weight="600">AWS path</text>

    <!-- CONVERGENCE BANNER -->
    <rect x="0" y="357" width="1060" height="18" fill="#0f2340"/>
    <text x="530" y="369" text-anchor="middle" font-size="9" font-weight="700" fill="rgba(255,255,255,0.85)" letter-spacing="0.1em">BOTH PATHS CONVERGE AT L4 — COMMON GATEWAY INFRASTRUCTURE</text>

    <!-- ═══ L4 LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="375" width="1060" height="118" fill="#f5f3fe"/>
    <rect x="0" y="375" width="90" height="118" fill="#7a72c8"/>
    <text x="45" y="424" text-anchor="middle" font-size="16" font-weight="700" fill="white">L4</text>
    <text x="45" y="440" text-anchor="middle" font-size="7.5" fill="#d8d4f4" letter-spacing="0.08em">APPLICATION</text>
    <text x="45" y="452" text-anchor="middle" font-size="7.5" fill="#d8d4f4" letter-spacing="0.08em">GATEWAY</text>

    <!-- Envoy box -->
    <rect x="98" y="382" width="556" height="104" fill="white" rx="5" stroke="#c8c0f0" stroke-width="1"/>
    <rect x="98" y="382" width="556" height="18" fill="#7a72c8" rx="5"/>
    <rect x="98" y="390" width="556" height="10" fill="#7a72c8"/>
    <text x="376" y="395" text-anchor="middle" font-size="11" font-weight="700" fill="white">Envoy Web Gateway</text>

    <!-- Internet auth chain -->
    <text x="108" y="413" font-size="8.5" fill="#7a8a9a" font-style="italic">Internet:</text>
    <rect x="162" y="406" width="68" height="16" fill="#eef3fa" rx="3" stroke="#c8d8f0" stroke-width="1"/>
    <text x="196" y="417" text-anchor="middle" font-size="8.5" fill="#1a3a6a">jwt_authn</text>
    <path d="M231,414 L240,414" stroke="#7a72c8" stroke-width="1" marker-end="url(#arrP)"/>
    <rect x="242" y="406" width="94" height="16" fill="#eef3fa" rx="3" stroke="#c8d8f0" stroke-width="1"/>
    <text x="289" y="417" text-anchor="middle" font-size="8.5" fill="#1a3a6a">Session Val (DPoP)</text>
    <path d="M337,414 L346,414" stroke="#7a72c8" stroke-width="1" marker-end="url(#arrP)"/>
    <rect x="348" y="406" width="68" height="16" fill="#eef3fa" rx="3" stroke="#c8d8f0" stroke-width="1"/>
    <text x="382" y="417" text-anchor="middle" font-size="8.5" fill="#1a3a6a">OPA PDP</text>
    <path d="M417,414 L426,414" stroke="#7a72c8" stroke-width="1" marker-end="url(#arrP)"/>
    <rect x="428" y="406" width="88" height="16" fill="#eef3fa" rx="3" stroke="#c8d8f0" stroke-width="1"/>
    <text x="472" y="417" text-anchor="middle" font-size="8.5" fill="#1a3a6a">Context Propagator</text>
    <text x="524" y="417" font-size="8" fill="#8B1A1A" font-style="italic">internet</text>

    <!-- VAN auth chain -->
    <text x="108" y="433" font-size="8.5" fill="#7a8a9a" font-style="italic">VAN:</text>
    <rect x="162" y="426" width="68" height="16" fill="#e8f5f0" rx="3" stroke="#1a6a5a" stroke-width="1"/>
    <text x="196" y="437" text-anchor="middle" font-size="8.5" fill="#1a6a5a">mTLS cert</text>
    <path d="M231,434 L240,434" stroke="#1a6a5a" stroke-width="1" marker-end="url(#arrT)"/>
    <rect x="242" y="426" width="94" height="16" fill="#e8f5f0" rx="3" stroke="#1a6a5a" stroke-width="1"/>
    <text x="289" y="437" text-anchor="middle" font-size="8.5" fill="#1a6a5a">Session Val (mTLS)</text>
    <path d="M337,434 L346,434" stroke="#1a6a5a" stroke-width="1" marker-end="url(#arrT)"/>
    <rect x="348" y="426" width="68" height="16" fill="#e8f5f0" rx="3" stroke="#1a6a5a" stroke-width="1"/>
    <text x="382" y="437" text-anchor="middle" font-size="8.5" fill="#1a6a5a">OPA PDP</text>
    <path d="M417,434 L426,434" stroke="#1a6a5a" stroke-width="1" marker-end="url(#arrT)"/>
    <rect x="428" y="426" width="88" height="16" fill="#e8f5f0" rx="3" stroke="#1a6a5a" stroke-width="1"/>
    <text x="472" y="437" text-anchor="middle" font-size="8.5" fill="#1a6a5a">Context Propagator</text>
    <text x="524" y="437" font-size="8" fill="#1a6a5a" font-style="italic">VAN</text>

    <text x="108" y="456" font-size="8.5" fill="#9b59b6" font-style="italic">Config via gRPC ADS · Zero-downtime propagation · OPA policy branch per origin type</text>
    <text x="108" y="469" font-size="8.5" fill="#5a7a9a" font-style="italic">Both paths share the same gateway infrastructure — origin type determines auth mechanism and OPA policy branch</text>

    <!-- Kong box -->
    <rect x="662" y="382" width="390" height="104" fill="white" rx="5" stroke="#b8c8e8" stroke-width="1"/>
    <rect x="662" y="382" width="390" height="18" fill="#4a78b8" rx="5"/>
    <rect x="662" y="390" width="390" height="10" fill="#4a78b8"/>
    <text x="857" y="395" text-anchor="middle" font-size="11" font-weight="700" fill="white">Kong API Gateway</text>
    <text x="857" y="413" text-anchor="middle" font-size="10" fill="#5a7a9a">Consumer identity · Rate limiting · Request transformation</text>
    <text x="857" y="427" text-anchor="middle" font-size="10" fill="#5a7a9a">OAuth client lifecycle · Plugin chain</text>
    <text x="857" y="441" text-anchor="middle" font-size="10" fill="#5a7a9a">Internet: OAuth/DPoP · VAN: mTLS consumer identity</text>
    <text x="857" y="455" text-anchor="middle" font-size="9" fill="#4a78b8" font-style="italic">Programmatic / API clients · Both internet and VAN origin</text>
    <text x="857" y="469" text-anchor="middle" font-size="9" fill="#4a78b8" font-style="italic">Reconciled via Kong Admin API</text>
    <line x1="0" y1="493" x2="1060" y2="493" stroke="#d0dae8" stroke-width="0.5"/>

    <!-- ═══ L5 LAYER ═══════════════════════════════════════════════ -->
    <rect x="0" y="493" width="1060" height="90" fill="#f2f8f5"/>
    <rect x="0" y="493" width="90" height="90" fill="#4a78b8"/>
    <text x="45" y="530" text-anchor="middle" font-size="16" font-weight="700" fill="white">L5</text>
    <text x="45" y="546" text-anchor="middle" font-size="7.5" fill="#ccdcee" letter-spacing="0.08em">SERVICES</text>
    <text x="45" y="558" text-anchor="middle" font-size="7" fill="#ccdcee" letter-spacing="0.06em">BOTH PATHS</text>

    <!-- On-Prem group -->
    <rect x="98" y="499" width="378" height="76" fill="white" rx="5" stroke="#5a7aaa" stroke-width="1.5"/>
    <text x="287" y="513" text-anchor="middle" font-size="9" font-weight="700" fill="#1a3a6a" letter-spacing="0.06em">ON-PREMISES</text>
    <rect x="106" y="517" width="112" height="50" fill="#f4f7fb" rx="4" stroke="#c8d8f0" stroke-width="1"/>
    <text x="162" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#1a4a8a">GAP</text>
    <text x="162" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Global App Platform</text>
    <text x="162" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">Greenplum / Pega</text>
    <rect x="226" y="517" width="112" height="50" fill="#f4f7fb" rx="4" stroke="#c8d8f0" stroke-width="1"/>
    <text x="282" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#1a4a8a">GKP</text>
    <text x="282" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Google Kubernetes</text>
    <text x="282" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">On-prem clusters</text>
    <rect x="346" y="517" width="122" height="50" fill="#f4f7fb" rx="4" stroke="#c8d8f0" stroke-width="1"/>
    <text x="407" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#1a4a8a">VSI</text>
    <text x="407" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Virtual Server Inst.</text>
    <text x="407" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">Bare-metal adjacent</text>

    <!-- AWS group -->
    <rect x="484" y="499" width="568" height="76" fill="white" rx="5" stroke="#f0a020" stroke-width="1.5"/>
    <text x="768" y="513" text-anchor="middle" font-size="9" font-weight="700" fill="#8a5200" letter-spacing="0.06em">AMAZON WEB SERVICES</text>
    <rect x="492" y="517" width="126" height="50" fill="#fffdf5" rx="4" stroke="#f0c870" stroke-width="1"/>
    <text x="555" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#8a5200">EKS</text>
    <text x="555" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Elastic Kubernetes</text>
    <text x="555" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">Managed K8s</text>
    <rect x="626" y="517" width="126" height="50" fill="#fffdf5" rx="4" stroke="#f0c870" stroke-width="1"/>
    <text x="689" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#8a5200">ECS</text>
    <text x="689" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Elastic Container</text>
    <text x="689" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">Managed containers</text>
    <rect x="760" y="517" width="126" height="50" fill="#fffdf5" rx="4" stroke="#f0c870" stroke-width="1"/>
    <text x="823" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#8a5200">EC2</text>
    <text x="823" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Virtual machines</text>
    <text x="823" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">IaaS compute</text>
    <rect x="894" y="517" width="150" height="50" fill="#fffdf5" rx="4" stroke="#f0c870" stroke-width="1"/>
    <text x="969" y="537" text-anchor="middle" font-size="11" font-weight="700" fill="#8a5200">Lambda</text>
    <text x="969" y="551" text-anchor="middle" font-size="8.5" fill="#7a8a9a">Serverless functions</text>
    <text x="969" y="562" text-anchor="middle" font-size="8" fill="#9aaaba">Event-driven</text>

    <!-- OPA AuthZen note -->
    <text x="440" y="581" text-anchor="middle" font-size="9" fill="#4a8a62" font-style="italic">OPA fine-grained AuthZen API available per service · x-auth-* identity headers forwarded for both internet and VAN origin</text>
    <line x1="0" y1="585" x2="1060" y2="585" stroke="#d0dae8" stroke-width="0.5"/>

    <!-- ═══ PLATFORM SERVICES ═══════════════════════════════════════ -->
    <rect x="0" y="585" width="1060" height="55" fill="white" stroke="#d0dae8" stroke-width="0.5"/>
    <rect x="0" y="585" width="1060" height="18" fill="#7a8e99"/>
    <text x="530" y="598" text-anchor="middle" font-size="10" font-weight="700" fill="white" letter-spacing="0.08em">PLATFORM SERVICES</text>
    <rect x="8" y="607" width="190" height="26" fill="#f4f7fb" rx="4" stroke="#d0dae8" stroke-width="1"/>
    <text x="103" y="618" text-anchor="middle" font-size="9.5" font-weight="700" fill="#9b59b6">Session Manager</text>
    <text x="103" y="629" text-anchor="middle" font-size="8.5" fill="#5a7a9a">Active/active · Multi-region · CAEP · &lt;1s revoke</text>
    <rect x="204" y="607" width="166" height="26" fill="#f4f7fb" rx="4" stroke="#d0dae8" stroke-width="1"/>
    <text x="287" y="618" text-anchor="middle" font-size="9.5" font-weight="700" fill="#546e7a">Kafka</text>
    <text x="287" y="629" text-anchor="middle" font-size="8.5" fill="#5a7a9a">Session · Revocation · Risk signals</text>
    <rect x="376" y="607" width="190" height="26" fill="#f4f7fb" rx="4" stroke="#d0dae8" stroke-width="1"/>
    <text x="471" y="618" text-anchor="middle" font-size="9.5" font-weight="700" fill="#4a8a62">OPA PDP + SpiceDB (L5)</text>
    <text x="471" y="629" text-anchor="middle" font-size="8.5" fill="#5a7a9a">ABAC policy · ReBAC opt-in</text>
    <rect x="572" y="607" width="240" height="26" fill="#f4f7fb" rx="4" stroke="#d0dae8" stroke-width="1"/>
    <text x="692" y="618" text-anchor="middle" font-size="9.5" font-weight="700" fill="#4a78b8">Ingress Control Plane</text>
    <text x="692" y="629" text-anchor="middle" font-size="8.5" fill="#5a7a9a">Mgmt API · Registry · GitOps · WAF-as-code</text>
    <rect x="818" y="607" width="234" height="26" fill="#f4f7fb" rx="4" stroke="#d0dae8" stroke-width="1"/>
    <text x="935" y="618" text-anchor="middle" font-size="9.5" font-weight="700" fill="#2e7d32">Observability</text>
    <text x="935" y="629" text-anchor="middle" font-size="8.5" fill="#5a7a9a">OTEL · Dynatrace · End-to-end tracing · SLOs</text>
  </svg>
```

---

## Resilience Posture

Full detail in `resilience-posture.html`. Summary:

| Layer | Model | RTO |
|---|---|---|
| DNS | Active/active (self-hosted + Cloudflare NS) | Instant — both NS sets serve simultaneously |
| L1/L2 | Active/active CDN (Akamai + Cloudflare) | Instant — Anycast, no failover event |
| L3 | Multi-region (NA, EMEA, APAC) | ~30s — GTM DNS TTL-bound |
| L4 | Multi-pod, HPA autoscaling, sidecar isolation | Subsecond — Kubernetes self-healing |
| Session Manager | Active/active, Kafka-replicated | Zero session loss on regional failover |
| Auth (Sentry/AuthE) | Pending regional topology confirmation | TBC — gate before Phase 3 |

**Ultimate DR path:** Direct to PSaaS+/AWS WAF, bypassing all CDN layers. Manual DNS cutover. RTO: hours. Last resort only.

---

## Current Plan State (Pre-Rebuild)

The plan HTML (`ingress-project-plan-light.html`) reflects the **pre-strategic-pivot** scope (19 engineers, Akamai primary/CF standby, no VAN path). It needs to be rebuilt to reflect:
- Active/active CDN with WAF-as-code pipeline
- VAN ingress path (new squad or expanded Gateway Squad scope)
- Dec 31, 2026 delivery (35 weeks from Apr 28)
- Additional resourcing (estimated 28-32 engineers for required parallelism)

### Current Phase Timeline (pre-rebuild, for reference)
| Phase | Squad | Start | Duration |
|---|---|---|---|
| P0 Foundation | Gateway (3 eng) | Apr 28, 2026 | 5w |
| P1 Data Plane | Gateway (6 eng) | Jun 2, 2026 | 16w |
| P2 Authentication | Identity (5 eng) | Jun 2, 2026 | 13w |
| P4 Authorization | Policy (4 eng) | Jun 2, 2026 | 16w |
| P3 Session Mgmt | Identity (5 eng) | Sep 2, 2026 | 13w |
| P5 Control Plane | Gateway+Console (8 eng) | Sep 23, 2026 | 10w |
| P6 Observability | Observability (2 eng) | Dec 1, 2026 | 6w |
| P7 Hardening | All (19 eng) | Jan 13, 2027 | 10w |

**Est. go-live (pre-rebuild):** Mar 24, 2027

---

## Squad Structure (Pre-Rebuild, 19 Engineers)

| Squad | Engineers | Phases | Owns |
|---|---|---|---|
| Gateway Squad | 6 | P0, P1, P5 | Data plane, control plane backend, WAF config |
| Identity Squad | 5 | P2, P3 | AuthN, Session Manager, CAEP, revocation |
| Policy Squad | 4 | P4 | OPA PDP, ABAC, SpiceDB L5 pattern, AuthZen |
| Console Squad | 2 | P5 | DE Console UI, developer docs |
| Observability Squad | 2 | P6 | OTEL, Dynatrace, health endpoints |
| Release Squad | 19 (all) | P7 | Hardening, DR testing, traffic migration |

---

## LOB Adopters

| Entity | LOB | Gateway | Notes |
|---|---|---|---|
| JPMM | Markets | Envoy | Web/API routes |
| JPMDB | Payments | Envoy | Browser flows, DPoP |
| JPMA | Payments | Envoy | Browser flows, DPoP |
| PDP | Payments | Kong | Programmatic API clients |
| HRL | Payments | Envoy | High Risk Location — workforce only, network segmentation for high-risk regions (China, Turkey). Approved CIB patterns exist. HRL route attributes needed in registry schema |

All contacts, route counts, and target dates TBC — Gavin to confirm with each entity.

---

## Key Technology Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Language | Go | Native to Envoy/OPA/K8s ecosystem, static binary, goroutine concurrency model, minimal Docker footprint |
| Gateway (browser) | Envoy | ext_authz filter chain, gRPC ADS control plane, xDS (LDS/RDS/CDS/EDS), WebSocket/HTTP2 support |
| Gateway (API) | Kong | Consumer identity, rate limiting, plugin chain, OAuth client lifecycle |
| Auth | DPoP + PKCE | Phishing-resistant, token bound to client key pair, stolen tokens cryptographically unusable |
| VAN auth | mTLS | Certificate-bound identity for private circuit clients, no browser crypto required |
| Policy engine | OPA | PDP at L4 via Envoy ext_authz (PEP). ABAC with dynamic PIP attribute resolution |
| ReBAC | SpiceDB (L5 opt-in) | Not in L4 path. Policy Squad provides reference schema patterns for service teams |
| Session | Custom Session Manager | JWT carries identity only (no entitlement claims). Active/active, Kafka-replicated |
| Authorization API | IETF AuthZen | L5 services call OPA via AuthZen for fine-grained resource decisions |
| CDN | Akamai + Cloudflare (both active) | Anycast, WAF-as-code sync, no single vendor dependency |
| DNS | Self-hosted + Cloudflare NS | Active/active, zone sync, no single point of failure at DNS layer |
| Observability | OTEL + Dynatrace | W3C traceparent end-to-end, Akamai header correlation at CDN boundary |
| GitOps | Bitbucket + ArgoCD | Policy CI in pipelines, ArgoCD ApplicationSets for multi-cluster |
| Container base | RHEL 9 (firm-approved) | Check approved library list for FIPS-validated Go crypto (BoringCrypto build) |
| Cloud | GKP (on-prem) + AWS | Self-contained per cloud, no cross-cloud dependency in request path |

---

## Open Items / Gates

| Item | Phase gate | Notes |
|---|---|---|
| Sentry/AuthE regional topology | Before P3 | If centralised, auth fails on regional failover. Must be regional. |
| PIP endpoint and ABAC attribute schema | Before P4 OPA authoring | OPA policy rules depend on attribute structure from PIP team |
| SpiceDB TAC approval | Initiate in P4 | Not on critical path. Fallback: custom ReBAC on PostgreSQL (+6w) |
| GTM composite health check threshold design | P1 | Functional decision, not tuning. Define before GTM config begins |
| N+1 regional capacity validation | P7 | Each region must sustain SLOs under peer-region failure load |
| Regional failover simulation | P7 | Full-region DR test required before go-live |
| SEAL PTx process (PTB, PTD, PTO) | P0 start | Initiate day one. PTD/PTO can progress iteratively |
| FinOps tagging standards | P0 | Tag resources with team, LOB, environment, component from day one |
| LOB contacts, route counts, onboarding dates | Pre-adoption | TBC per entity. Not needed for platform go-live |
| WAF-as-code pipeline design | P1 (new scope) | CI/CD pushes identical ruleset to Akamai and Cloudflare APIs |
| VAN ingress path scope | Plan rebuild | New squad or expanded Gateway Squad. mTLS, OPA VAN policy branch, registry VAN route attributes |

---

## Files in This Project

| File | Description |
|---|---|
| `ingress-project-plan-light.html` | Full project plan — phases, deliverables, Gantt charts, architecture diagram, LOB adoption. **Pre-rebuild (old scope)** |
| `resilience-posture.html` | Resilience posture one-pager — layer-by-layer with RTOs, ultimate DR path, open gates |
| `generate-gantt-light.js` | Node.js generator script for the project plan HTML. Source of truth for plan content |
| `CLAUDE.md` | This file — project context for Claude Code |

---

## Coding Conventions

- **Language:** Go for all platform services
- **Style:** Standard Go conventions, `gofmt`, no exceptions
- **Module structure:** Shared OTEL library, shared DPoP verification library, shared health check contract — published to internal artifact registry
- **Container:** RHEL 9 base image from firm registry. Non-root user. Static Go binary
- **CI/CD:** Bitbucket Pipelines for build/test/policy CI. ArgoCD ApplicationSets for multi-cluster deploy
- **Observability:** Every service instrumented with shared OTEL Go library. W3C traceparent on every request. 100% sampling in production
- **Health probes:** `/health` (liveness) and `/health/detail` (readiness) on every service. Readiness checks JWKS loaded, OPA bundle loaded, Kafka connected, DB reachable

---

## Glossary

| Term | Definition |
|---|---|
| ADS | Aggregated Discovery Service — Envoy's gRPC-based control plane protocol |
| ABAC | Attribute-Based Access Control — dynamic policy evaluation against user/resource attributes |
| CAEP | Continuous Access Evaluation Protocol — IETF spec for real-time session revocation events |
| CTC Edge | JPM's AWS-hosted perimeter layer |
| DE-TAC | Digital Enablement Technology Architecture Committee |
| DPoP | Demonstrating Proof of Possession — RFC 9449, token bound to client's ephemeral key pair |
| EDS/CDS/LDS/RDS | Envoy xDS resource types: Endpoint/Cluster/Listener/Route Discovery Service |
| ext_authz | Envoy external authorization filter — PEP that calls OPA PDP per request |
| HRL | High Risk Location — JPM network segmentation for workforce users in restricted regions |
| PDP | Policy Decision Point — OPA evaluates requests and returns allow/deny |
| PEP | Policy Enforcement Point — Envoy ext_authz enforces OPA decisions |
| PIP | Policy Information Point — ABAC attribute source called by OPA at request time |
| PSaaS+ | JPM's on-premises perimeter/proxy layer (GKP path) |
| ReBAC | Relationship-Based Access Control — graph-based (SpiceDB) |
| SEAL PTx | JPM internal governance process: Permit To Build, Permit To Deploy, Permit To Operate |
| VAN | Value Added Network — private leased line connectivity for institutional clients |
| WAF-as-code | GitOps-managed WAF ruleset with CI/CD push to both Akamai and Cloudflare APIs |
| xDS | Envoy's extensible Discovery Service protocol family (LDS, RDS, CDS, EDS) |
