# OBO (On-Behalf-Of) in the Context of AI Agent Permission Delegation

## Executive Summary

On-Behalf-Of (OBO) is emerging as a foundational pattern for AI agent authorization. Originally defined in OAuth 2.0 Token Exchange (RFC 8693) for service-to-service delegation, OBO is now being actively extended for AI agents through IETF Internet-Drafts, platform-specific implementations (Microsoft Entra Agent ID), and complementary protocols (Google A2A, MCP, GNAP). This report provides a deep technical analysis of how OBO mechanisms are being adapted to address the unique challenges of AI agent permission delegation — including multi-hop delegation chains, scope attenuation, the confused deputy problem, and auditability — and surveys the emerging standards landscape as of early 2026.

## Table of Contents

1. [Foundations: RFC 8693 and Traditional OBO](#1-foundations-rfc-8693-and-traditional-obo)
2. [The IETF Draft: OAuth 2.0 OBO Extension for AI Agents](#2-the-ietf-draft-oauth-20-obo-extension-for-ai-agents)
3. [Microsoft Entra Agent ID: OBO in Practice](#3-microsoft-entra-agent-id-obo-in-practice)
4. [Delegation Token Mechanics: Claims, Chains, and Composites](#4-delegation-token-mechanics-claims-chains-and-composites)
5. [Multi-Agent Delegation Patterns](#5-multi-agent-delegation-patterns)
6. [Complementary Protocols and Standards](#6-complementary-protocols-and-standards)
7. [Alternative Token Formats: Macaroons, Biscuits, and Offline Attenuation](#7-alternative-token-formats-macaroons-biscuits-and-offline-attenuation)
8. [Security Considerations and Open Problems](#8-security-considerations-and-open-problems)
9. [Conclusions](#9-conclusions)
10. [Sources](#10-sources)

---

## 1. Foundations: RFC 8693 and Traditional OBO

### 1.1 OAuth 2.0 Token Exchange

<a href="https://datatracker.ietf.org/doc/html/rfc8693" target="_blank">RFC 8693</a> defines a protocol for requesting and obtaining security tokens from OAuth 2.0 authorization servers, establishing the foundational semantics for OBO delegation. The specification introduces a Security Token Service (STS) model using the grant type `urn:ietf:params:oauth:grant-type:token-exchange`.

### 1.2 Delegation vs. Impersonation

RFC 8693 draws a critical distinction between two authorization semantics:

**Impersonation**: Principal A receives all rights of Principal B and becomes indistinguishable from B. The resource server sees only B's identity. Use cases include API gateways exchanging tokens for legacy systems or support staff reproducing user issues.

**Delegation**: Principal A retains its own identity while explicitly acting on behalf of B. Both identities are preserved in a "composite token." As the RFC states: "principal A still has its own identity separate from B, and it is explicitly understood that while B may have delegated some of its rights to A, any actions taken are being taken by A representing B."

This distinction is fundamental for AI agents: delegation (not impersonation) is the correct semantic, because resource servers and audit systems must know *both* who authorized an action (the user) and *who performed it* (the agent).

### 1.3 Request Parameters

The token exchange request uses these key parameters:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `subject_token` | Yes | Represents the party on whose behalf the request is made |
| `subject_token_type` | Yes | Identifies the subject token's format (JWT, SAML, etc.) |
| `actor_token` | No | Represents the acting party authorized to use the issued token |
| `actor_token_type` | Conditional | Required when `actor_token` is present |
| `scope` | No | Requested scope for the new token |
| `audience` | No | Target service for the new token |

### 1.4 The `act` and `may_act` Claims

RFC 8693 introduces two JWT claims specifically for delegation:

- **`act` (actor)**: Identifies the acting party in a delegation scenario. It "provides a means within a JWT to express that delegation has occurred and identify the acting party to whom authority has been delegated."

- **`may_act` (authorized actor)**: An authorization assertion embedded in the subject token stating that a particular party is pre-authorized to act on the subject's behalf. This allows authorization servers to validate delegation requests without additional out-of-band checks.

---

## 2. The IETF Draft: OAuth 2.0 OBO Extension for AI Agents

### 2.1 Overview

The most significant standards effort targeting AI agent OBO is <a href="https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/" target="_blank">draft-oauth-ai-agents-on-behalf-of-user</a> (currently at version 02), authored by T. S. Senarath and A. Dissanayaka from WSO2. This Internet-Draft extends the OAuth 2.0 Authorization Code Grant flow to enable user-delegated authorization specifically for AI agents.

### 2.2 Why a New Extension?

Standard OAuth 2.0 flows do not fully address agent delegation:

- **Authorization Code Grant**: Identifies the client application but not the specific agent acting within it. The user consents to the *application*, not the *agent*.
- **Client Credentials Grant**: Provides no user consent mechanism — the agent operates under application-level privileges.
- **RFC 8693 Token Exchange**: Lacks native support for obtaining explicit user consent for an agent via the front channel. It does not specify how to acquire the initial subject token through a user-interactive flow.

### 2.3 New Parameters

The draft introduces two parameters:

**`requested_actor`** (Authorization Endpoint): The unique identifier of the AI agent for which the client is requesting delegated access. This parameter is REQUIRED in the authorization request and allows the authorization server to present agent-specific consent screens.

**`actor_token`** (Token Endpoint): A valid token issued to the agent, authenticating the agent during the authorization code exchange. It MUST include a `sub` claim identifying the agent.

### 2.4 Protocol Flow

The full ten-step flow:

```
┌──────┐  ┌──────────┐  ┌────────────┐  ┌─────────────┐  ┌──────────────┐
│ User │  │ AI Agent │  │   Client   │  │  AuthZ Svr  │  │Resource Svr  │
└──┬───┘  └────┬─────┘  └─────┬──────┘  └──────┬──────┘  └──────┬───────┘
   │           │              │                 │                │
   │           │   1. Agent signals need to act │                │
   │           │──────────────>                 │                │
   │           │              │  2. Access resource              │
   │           │              │─────────────────────────────────>│
   │           │              │  3. 401/403 + WWW-Authenticate   │
   │           │              │<─────────────────────────────────│
   │  4. Redirect with requested_actor + PKCE   │                │
   │<─────────────────────────│                 │                │
   │  5. Authenticate + consent for agent+scopes│                │
   │─────────────────────────────────────────── >                │
   │           │              │  6. Authorization code           │
   │           │              │<────────────────│                │
   │           │              │  7. Code + PKCE verifier +       │
   │           │              │     actor_token                  │
   │           │              │────────────────>│                │
   │           │              │  8. Validate code, PKCE,         │
   │           │              │     actor_token                  │
   │           │              │  9. Issue OBO access token       │
   │           │              │<────────────────│                │
   │           │              │  10. Access resource with OBO    │
   │           │              │      token                       │
   │           │              │─────────────────────────────────>│
```

Key innovations:
- The flow can be **triggered by a resource server challenge** (HTTP 401/403), enabling dynamic consent acquisition.
- The authorization code is bound to the triple of (user, client, actor), not just (user, client).
- PKCE is REQUIRED for all flows.

### 2.5 Access Token Claims

The issued JWT contains:

```json
{
  "iss": "https://authorization-server.com/oauth2/token",
  "sub": "user-456",
  "azp": "s6BhdRkqt3",
  "act": { "sub": "actor-finance-v1" },
  "scope": "read:email write:calendar",
  "exp": 1746009896,
  "iat": 1745923496,
  "jti": "unique-token-id"
}
```

The `sub` claim identifies the user. The `act.sub` claim identifies the agent. The `azp` claim identifies the client application. This provides a **clear, auditable delegation chain**: User → Client → Agent → Resource.

### 2.6 Related IETF Drafts

Several other drafts address adjacent aspects of AI agent authorization:

- **<a href="https://datatracker.ietf.org/doc/draft-klrc-aiagent-auth/" target="_blank">draft-klrc-aiagent-auth-00</a>**: A comprehensive framework composing SPIFFE/WIMSE identifiers, mTLS, and OAuth 2.0 for agent authentication and authorization. Defines an Agent Identity Management System (AIMS) with layered attestation.
- **<a href="https://www.ietf.org/archive/id/draft-rosenberg-oauth-aauth-00.html" target="_blank">draft-rosenberg-oauth-aauth-00 (AAuth)</a>**: An OAuth 2.1 extension for AI agents that collect PII from users over voice/text channels, obtaining tokens through user identity verification rather than redirect flows.
- **<a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">draft-liu-oauth-a2a-profile-00</a>**: An A2A Profile for OAuth Transaction Tokens that embeds agent-to-agent call chain context within short-lived transaction tokens.
- **<a href="https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/" target="_blank">draft-ietf-oauth-identity-chaining-08</a>**: Defines mechanisms for preserving identity and authorization across trust domain boundaries — critical for multi-cloud agent deployment.

---

## 3. Microsoft Entra Agent ID: OBO in Practice

### 3.1 Agent Identity Architecture

<a href="https://learn.microsoft.com/en-us/entra/agent-id/identity-platform/agent-on-behalf-of-oauth-flow" target="_blank">Microsoft Entra Agent ID</a> is the first major platform implementation of OBO for AI agents, introducing two key concepts:

- **Agent Identity Blueprint**: A parent application registration that defines the agent's capabilities and delegated permissions. It acts as an OAuth resource (API) application.
- **Agent Identity**: A child instance of the blueprint, representing a specific running agent. It inherits delegated permissions from its parent via the `InheritDelegatedPermissions` property.

### 3.2 Protocol Steps

The flow extends standard OAuth 2.0 OBO with a managed identity credential layer:

1. **User authenticates** with the client and obtains an access token (Tc).
2. **Client sends Tc** to the Agent Identity Blueprint.
3. **Agent Identity Blueprint** requests an exchange token by presenting its managed identity credential (TUAMI):
   ```
   POST /oauth2/v2.0/token
   client_id=AgentBlueprint
   &scope=api://AzureADTokenExchange/.default
   &fmi_path=AgentIdentity
   &client_assertion=TUAMI
   &grant_type=client_credentials
   ```
   This returns T1, a token identifying the agent.
4. **Agent Identity** performs the OBO exchange using both T1 and Tc:
   ```
   POST /oauth2/v2.0/token
   client_id=AgentIdentity
   &scope=https://resource.example.com/scope1
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion={T1}
   &grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion={Tc}
   &requested_token_use=on_behalf_of
   ```
5. **Entra ID validates** both tokens (T1 audience must match the Agent Identity Blueprint client ID) and issues a resource token.

### 3.3 Key Design Decisions

- **Managed identities** are the preferred credential type, providing automatic rotation and secure storage.
- **Client secrets are explicitly discouraged** in production for agent identity blueprints.
- **Permission inheritance** across blueprint-to-identity boundaries reduces consent complexity for multi-instance deployments, but is bounded within tenant limits.
- **Refresh token support** enables asynchronous and background processing scenarios.

---

## 4. Delegation Token Mechanics: Claims, Chains, and Composites

### 4.1 Composite Token Structure

A delegation token — or "composite token" — encodes both the subject (user) and the actor (agent). The basic structure:

```json
{
  "sub": "user@example.com",
  "act": {
    "sub": "agent:infrabot"
  },
  "aud": "api.github.com",
  "scope": "read:repos",
  "exp": 1721890041
}
```

The `sub` claim answers "on whose authority?" The `act.sub` claim answers "who is acting?" The `aud` claim restricts where the token is valid. The `scope` limits what actions are permitted.

### 4.2 Nested `act` Claims for Multi-Hop Chains

When agents delegate to other agents, the `act` claim nests recursively:

```json
{
  "sub": "user:sam",
  "act": {
    "sub": "agent:market-analysis",
    "act": {
      "sub": "agent:supply-chain",
      "act": {
        "sub": "agent:data-retriever"
      }
    }
  }
}
```

Per RFC 8693: "the outermost `act` claim represents the current actor while nested `act` claims represent prior actors." Critically, **for access control decisions, only the top-level claims and the current (outermost) actor should be considered**. Nested actors serve as an audit trail, not an authorization input.

### 4.3 SPIFFE-Based Actor Identification

In cloud-native deployments, actor identifiers can use <a href="https://spiffe.io/docs/latest/spire-about/spire-concepts/" target="_blank">SPIFFE</a> URIs, providing attestation-bound identity:

```json
{
  "act": {
    "sub": "spiffe://cluster.local/ns/default/sa/market-analysis-agent",
    "act": {
      "sub": "spiffe://cluster.local/ns/default/sa/supply-chain-agent"
    }
  },
  "act_depth": 2,
  "sub": "user-uuid-here",
  "typ": "OBO"
}
```

SPIFFE provides workload attestation (identity derived from execution environment, not mere registration), short-lived credentials (SVIDs), and cross-trust-domain federation — all critical properties for agent identity at scale.

---

## 5. Multi-Agent Delegation Patterns

### 5.1 Scope Attenuation

A fundamental principle of OBO delegation is that **each hop must narrow, never expand, permissions**. The resulting token represents the intersection of:

1. The user's permissions
2. The agent/service's configured permissions
3. The target resource's acceptance policies

As <a href="https://blog.christianposta.com/explaining-on-behalf-of-for-ai-agents/" target="_blank">Christian Posta explains</a>, token exchange ensures "authority reduction, not amplification" — each hop produces more constrained tokens with specific audiences and narrowed scope.

Example attenuation pattern:
```
User Token:        {identity, expiry, buy_stock, market_data, email}
  ↓ Agent A OBO:   {identity, expiry, buy_stock, market_data}  [email removed]
    ↓ Agent B OBO: {identity, expiry, market_data}              [buy_stock removed]
```

### 5.2 Token Broker Architecture

<a href="https://www.garvik.dev/ai/on-behalf-of-token-complex" target="_blank">Enterprise implementations</a> typically introduce a Token Broker intermediary:

1. User authenticates and receives a standard access token.
2. Application exchanges this token with the Token Broker for a scoped OBO token.
3. AI Agent receives the narrower, time-limited OBO token — never the original user token.
4. MCP Server or resource server validates the OBO token before executing any action.

This prevents "God Mode" scenarios where agents inherit all user permissions across services.

### 5.3 Transaction Tokens for Call Chains

The <a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">A2A Profile for OAuth Transaction Tokens</a> proposes embedding call chain context in short-lived tokens:

- **`tctx` (transaction context)**: Immutable values from the original user input, preserved unchanged throughout the chain.
- **`rctx` (request context)**: Mutable context that agents append at each hop (observations, decisions, metadata).
- **`purp` (purpose)**: A task identifier linking all tokens in a delegation chain.

These transaction tokens provide "least-privilege guarantees and auditability" through sub-minute lifetimes and immutable call context that prevents scope expansion.

---

## 6. Complementary Protocols and Standards

### 6.1 Google Agent2Agent (A2A) Protocol

The <a href="https://a2a-protocol.org/latest/specification/" target="_blank">A2A protocol</a> enables interoperable agent communication but takes a different approach to delegation than OBO:

- **Agent Cards** declare supported authentication schemes (API Key, OAuth 2.0, OIDC, mTLS).
- **`AUTH_REQUIRED` task state** enables agents to delegate authorization responsibility back to the client when additional permissions are needed.
- A2A does **not** implement direct OBO patterns for agent-to-agent delegation. Authentication remains client-to-server.
- <a href="https://github.com/a2aproject/A2A/issues/19" target="_blank">Issue #19</a> on the A2A repository explicitly tracks "Delegated User Authorization for Agent2Agent Servers" as an open design question.

### 6.2 Model Context Protocol (MCP)

<a href="https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization" target="_blank">MCP's authorization model</a> mandates OAuth 2.1 with OIDC for remote servers:

- Token validation occurs at **every tool invocation**, not just session establishment.
- MCP servers can act as OAuth clients to third-party authorization servers while acting as OAuth authorization servers to MCP clients — a dual-role delegation pattern.
- The multi-hop chain (user → AI host → MCP client → MCP server → backend API) must preserve authorization context at each boundary.
- <a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">MCP explicitly uses OBO</a> to prevent the confused deputy problem: rather than forwarding user tokens, the MCP server exchanges them for server-issued OBO tokens scoped to the specific operation.

### 6.3 GNAP (RFC 9635)

<a href="https://www.rfc-editor.org/rfc/rfc9635.html" target="_blank">GNAP (Grant Negotiation and Authorization Protocol)</a>, finalized as RFC 9635 in October 2024, offers architectural advantages over OAuth 2.0 for agent delegation:

- **Role separation**: GNAP splits the resource owner from the end user, allowing clearer modeling of "agent requests access that user owns."
- **No pre-registration required**: Agents don't need to be registered as OAuth clients in advance — they negotiate grants dynamically.
- **Multiple access tokens per grant**: A single grant request can yield tokens for multiple resources.
- **Continuing grants**: Long-lived authorization relationships can be renegotiated over time without re-consent.
- **JSON-native**: Structured request/response bodies (vs. OAuth's form-encoded parameters) are better suited to machine-to-machine interactions.

However, GNAP adoption remains nascent — <a href="https://ciamweekly.substack.com/p/gnap-for-mcp-authorization" target="_blank">approximately ten known implementations exist</a> as of early 2026.

### 6.4 WIMSE (Workload Identity in Multi-System Environments)

The <a href="https://datatracker.ietf.org/wg/wimse/about/" target="_blank">IETF WIMSE working group</a> addresses workload identity across clouds and platforms. Recent drafts have added explicit AI agent use cases:

- <a href="https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/" target="_blank">draft-ietf-wimse-arch-07</a> now includes AI-specific scenarios, SPIFFE-based agent identifiers, and alignment with OAuth identity chaining.
- WIMSE workload credentials provide the cryptographic identity layer that OBO tokens reference in the `act` claim.

### 6.5 AuthZEN (OpenID Authorization API 1.0)

<a href="https://openid.net/wg/authzen/" target="_blank">AuthZEN</a>, approved as a Final Specification in January 2026, standardizes authorization evaluation APIs. For AI agents, every tool invocation maps to an AuthZEN evaluation:

- **Subject**: The agent's identity (e.g., SPIFFE ID)
- **Action**: The operation (e.g., "send_email")
- **Resource**: The target (e.g., "contact_list")
- **Context**: The delegating user, blast radius classification, reversibility flag, and behavioral anomaly score

AuthZEN fills a gap that <a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">authentication standards leave open</a>: per-action authorization evaluation rather than scope-at-issuance enforcement.

---

## 7. Alternative Token Formats: Macaroons, Biscuits, and Offline Attenuation

### 7.1 The Limitation of JWT-Based OBO

Standard JWT tokens cannot be restricted after issuance without contacting the authorization server. In decentralized multi-agent systems, this creates latency and single-point-of-failure risks at scale.

### 7.2 Macaroons

<a href="https://research.google/pubs/pub41892/" target="_blank">Macaroons</a> (Google Research, 2014) are authorization credentials with append-only "caveats" that attenuate permissions without server validation:

- Token holders can add signed constraints locally, never expanding permissions.
- Caveats create verifiable chains traceable to the original issuer.
- **Macaroons are the only credential primitive where the holder can reduce their own authority and pass it downstream** — exactly what agent delegation requires.

### 7.3 Biscuits

<a href="https://www.clever.cloud/blog/engineering/2021/04/12/introduction-to-biscuit/" target="_blank">Biscuits</a> improve on Macaroons by using public key cryptography (vs. MAC-based shared secrets), enabling verification without shared secrets across trust domains.

### 7.4 Delegation Capability Tokens (DCTs)

<a href="https://www.marktechpost.com/2026/02/15/google-deepmind-proposes-new-framework-for-intelligent-ai-delegation-to-secure-the-emerging-agentic-web-for-future-economies/" target="_blank">Google DeepMind proposes</a> Delegation Capability Tokens (DCTs) based on Macaroon/Biscuit technologies, using cryptographic caveats to enforce least privilege in agent delegation chains. The <a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Okta analysis</a> illustrates the pattern:

```
Primary Agent Token: {identity, expiry, buy_stock, market_data}
  ↓ (add caveat)
Sub-Agent Token:     {identity, expiry, market_data}  [buy_stock removed]
```

Each delegation adds signed constraints; the token remains verifiable offline while preventing unauthorized capability expansion.

---

## 8. Security Considerations and Open Problems

### 8.1 The Confused Deputy Problem

The <a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">confused deputy</a> is the most significant IAM threat in agentic systems. It occurs when an agent with elevated privileges is tricked into performing unauthorized actions on a user's behalf. OBO mitigates this by:

- Ensuring the agent never holds the user's full credentials.
- Binding each action to both agent and user identity.
- Requiring cryptographic proof of the current user session (as implemented by <a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Auth0 Token Vault</a>).

### 8.2 Agent Session Smuggling and Cross-Agent Privilege Escalation

<a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Okta identifies</a> two novel attack vectors:

- **Agent session smuggling**: A sub-agent embeds hidden instructions in responses. The parent agent executes these covert commands without user visibility.
- **Cross-agent privilege escalation**: Compromised agents reconfigure sibling agents' runtime environments, creating self-reinforcing control loops.

### 8.3 Missing Cryptographic Lineage

Current OAuth token exchange validates token structure but lacks historical traceability. By the third delegation hop, **no cryptographic link exists to the initiating agent or user**. The `act` claim shows who is acting, but not whether the chain was authorized. This is a fundamental gap that Macaroon/Biscuit-style tokens address through append-only caveat chains.

### 8.4 The Agency vs. Privilege Distinction

<a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">OWASP's agentic security framework</a> distinguishes between:

- **Least privilege**: Controlling *what* resources an agent accesses (addressed by OBO scoping)
- **Least agency**: Constraining *how autonomously* the agent acts within permitted access (largely unaddressed by current standards)

An agent holding `email:send` scope can send one message or thousands — current OBO standards treat both identically until token expiration.

### 8.5 Practical Mitigations

| Mitigation | Mechanism | Standard/Implementation |
|-----------|-----------|------------------------|
| Scope attenuation | Narrow permissions at each hop | RFC 8693, Macaroons |
| Short-lived tokens | 5-15 minute lifetimes | All OBO implementations |
| Audience restriction | Tokens valid for single resource | RFC 8693 `aud` claim |
| Out-of-band confirmation | Sensitive actions require approval via separate channel | Okta Async AuthZ |
| Per-action evaluation | Evaluate authorization at each tool invocation | AuthZEN |
| DPoP binding | Cryptographically bind tokens to agent keys | OAuth DPoP |
| Transaction tokens | Sub-minute tokens bound to specific transactions | draft-liu-oauth-a2a-profile |
| Delegation depth limits | Reject chains exceeding N hops | Implementation-specific |

---

## 9. Conclusions

OBO for AI agents is rapidly evolving from a service-to-service pattern into a first-class agent authorization primitive. The key developments as of early 2026:

1. **Standards are converging**: The IETF draft (draft-oauth-ai-agents-on-behalf-of-user) provides a user-consent-driven OBO flow purpose-built for AI agents, complemented by draft-klrc-aiagent-auth for agent identity and draft-liu-oauth-a2a-profile for transaction tokens.

2. **Production implementations exist**: Microsoft Entra Agent ID demonstrates a working OBO flow with managed identities, permission inheritance, and refresh token support for async agents.

3. **Fundamental gaps remain**: Per-action authorization (solved by AuthZEN), cryptographic lineage (solved by Macaroons/Biscuits), and agency constraints (largely unsolved) represent the frontier.

4. **The ecosystem is fragmented**: A2A, MCP, GNAP, and various IETF drafts each address parts of the problem without full interoperability. WIMSE and OAuth identity chaining are bridging cross-domain gaps.

5. **Scope attenuation is the core principle**: Whether through JWT claim filtering, Macaroon caveats, or token broker architectures, the pattern is consistent — each delegation hop must narrow, never expand, the agent's authority.

The trajectory is clear: OBO will become a mandatory capability for any agent framework operating in enterprise contexts. Organizations deploying AI agents today should implement OBO token exchange (RFC 8693) with explicit agent identity in the `act` claim, enforce short-lived tokens with audience restrictions, and plan for AuthZEN-style per-action evaluation as the authorization layer matures.

---

## 10. Sources

1. <a href="https://datatracker.ietf.org/doc/html/rfc8693" target="_blank">RFC 8693 - OAuth 2.0 Token Exchange</a>
2. <a href="https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/" target="_blank">draft-oauth-ai-agents-on-behalf-of-user-02 - OAuth 2.0 Extension: On-Behalf-Of User Authorization for AI Agents</a>
3. <a href="https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html" target="_blank">draft-oauth-ai-agents-on-behalf-of-user-01 (full text)</a>
4. <a href="https://learn.microsoft.com/en-us/entra/agent-id/identity-platform/agent-on-behalf-of-oauth-flow" target="_blank">Microsoft Entra Agent ID - On-Behalf-Of Flow</a>
5. <a href="https://a2a-protocol.org/latest/specification/" target="_blank">Agent2Agent (A2A) Protocol Specification</a>
6. <a href="https://github.com/a2aproject/A2A/issues/19" target="_blank">A2A Issue #19: Delegated User Authorization for Agent2Agent Servers</a>
7. <a href="https://www.rfc-editor.org/rfc/rfc9635.html" target="_blank">RFC 9635 - Grant Negotiation and Authorization Protocol (GNAP)</a>
8. <a href="https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization" target="_blank">MCP Authorization Specification</a>
9. <a href="https://openid.net/wg/authzen/" target="_blank">AuthZEN Working Group - OpenID Foundation</a>
10. <a href="https://datatracker.ietf.org/doc/draft-klrc-aiagent-auth/" target="_blank">draft-klrc-aiagent-auth-00 - AI Agent Authentication and Authorization</a>
11. <a href="https://www.ietf.org/archive/id/draft-rosenberg-oauth-aauth-00.html" target="_blank">draft-rosenberg-oauth-aauth-00 - AAuth: Agentic Authorization OAuth 2.1 Extension</a>
12. <a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">draft-liu-oauth-a2a-profile-00 - A2A Profile for OAuth Transaction Tokens</a>
13. <a href="https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/" target="_blank">draft-ietf-oauth-identity-chaining-08 - OAuth Identity and Authorization Chaining Across Domains</a>
14. <a href="https://datatracker.ietf.org/wg/wimse/about/" target="_blank">WIMSE Working Group - IETF</a>
15. <a href="https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/" target="_blank">draft-ietf-wimse-arch-07 - WIMSE Architecture</a>
16. <a href="https://blog.christianposta.com/explaining-on-behalf-of-for-ai-agents/" target="_blank">Explaining OAuth Delegation, 'On Behalf Of', and Agent Identity for AI Agents - Christian Posta</a>
17. <a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Control the Chain, Secure the System: Fixing AI Agent Delegation - Okta</a>
18. <a href="https://arxiv.org/html/2501.09674v1" target="_blank">Authenticated Delegation and Authorized AI Agents - arXiv</a>
19. <a href="https://www.scalekit.com/blog/delegated-agent-access" target="_blank">On-Behalf-Of Authentication for AI Agents - Scalekit</a>
20. <a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">MCP Authentication and Authorization: OAuth 2.1, Token Delegation, and the Confused Deputy Problem - FlowHunt</a>
21. <a href="https://www.strata.io/blog/agentic-identity/why-agentic-ai-demands-more-from-oauth-6a/" target="_blank">2026 Guide to OAuth Token Exchange & Agentic AI - Strata</a>
22. <a href="https://www.garvik.dev/ai/on-behalf-of-token-complex" target="_blank">Deep Dive: Enterprise Security with OBO Tokens & AI Agents - Garvik</a>
23. <a href="https://research.google/pubs/pub41892/" target="_blank">Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud - Google Research</a>
24. <a href="https://www.clever.cloud/blog/engineering/2021/04/12/introduction-to-biscuit/" target="_blank">Biscuit, the Foundation for Your Authorization Systems - Clever Cloud</a>
25. <a href="https://spiffe.io/docs/latest/spire-about/spire-concepts/" target="_blank">SPIFFE/SPIRE Concepts</a>
26. <a href="https://ciamweekly.substack.com/p/gnap-for-mcp-authorization" target="_blank">GNAP For MCP Authorization - CIAM Weekly</a>
27. <a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">AI Agent Authentication Gets the Hard Part Right. Authorization Is Still Your Problem.</a>
28. <a href="https://dev.to/kanywst/rfc-8693-deep-dive-token-exchange-310i" target="_blank">RFC 8693 Deep Dive: Token Exchange - DEV Community</a>
29. <a href="https://medium.com/@shashimalsenarath.17/how-ai-agents-can-safely-act-on-your-behalf-a-new-oauth-2-0-extension-81ca4c3e53b8" target="_blank">How AI Agents Can Safely Act on Your Behalf: A New OAuth 2.0 Extension - Medium</a>
30. <a href="https://www.marktechpost.com/2026/02/15/google-deepmind-proposes-new-framework-for-intelligent-ai-delegation-to-secure-the-emerging-agentic-web-for-future-economies/" target="_blank">Google DeepMind Proposes New Framework for Intelligent AI Delegation - MarkTechPost</a>
