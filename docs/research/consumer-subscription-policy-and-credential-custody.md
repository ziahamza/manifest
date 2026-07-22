# Consumer subscription policy and credential-custody constraints

**Research snapshot:** 2026-07-22

**Scope:** A self-hosted Manifest gateway that stores, refreshes, isolates, and selects among Claude Code and ChatGPT/Codex credentials.

**Status:** Architecture constraint, not legal advice. Public vendor terms can be superseded by a customer's signed order form or negotiated agreement; counsel and the vendor must confirm any exception in writing.

## Decision

Manifest must **not pool, load-balance, or fail over across consumer or per-seat subscription credentials**. A credential may not become fungible capacity for a different person, tenant, or workload after its owner reaches a quota. That behavior would make an account available to someone else and/or configure the service to avoid provider usage limits.

The supported production foundation is:

1. **Shared products and shared automation use provider API credentials or workload identities** owned by the organization running the gateway.
2. **Subscription access remains bound to one authenticated human and their entitled workspace.** It may be used only through a first-party client or another flow the provider explicitly documents for that exact use; it cannot be placed in a cross-user pool.
3. **Anthropic Free, Pro, and Max credentials cannot be routed by Manifest on users' behalf.** Anthropic expressly disallows third-party developers offering Claude.ai login or routing those plan credentials for users.
4. **Anthropic Team/Enterprise and OpenAI Business/Enterprise subscription gateway use requires a written vendor answer or negotiated agreement before implementation.** Their public materials support per-user seats and certain trusted automation, but do not grant a right to pool seat credentials.
5. **No quota-evasion fallback.** On subscription exhaustion, return the provider limit and its reset information or move to an explicitly authorized, separately billed API lane. Never silently borrow another person's subscription.

This conclusion blocks the proposed “many consumer subscriptions as one capacity pool” data plane. It does not block Manifest as a control plane over API organizations, supported cloud providers, enterprise workload credentials, or owner-bound native clients.

## Why subscription pooling is not an acceptable foundation

### Anthropic: Free, Pro, and Max are an explicit no

Anthropic's Consumer Terms apply to Claude Free, Pro, and Max. They prohibit sharing account login information, API keys, or account credentials; making an account available to another person; and automated/non-human access except through an Anthropic API key or where Anthropic explicitly permits it. See [Consumer Terms sections 2 and 3](https://www.anthropic.com/legal/consumer-terms).

Anthropic's Claude Code legal guidance is more specific: OAuth is for ordinary use of native Anthropic applications, developers building products should use an API key or supported cloud provider, and third-party developers may not offer Claude.ai login or route requests through Free, Pro, or Max credentials on behalf of users. Anthropic reserves enforcement without prior notice. See [Claude Code legal and compliance: Authentication and credential use](https://code.claude.com/docs/en/legal-and-compliance).

The account-login guidance also says subscription usage is intended for subscribers' ordinary use of native Anthropic applications. It prohibits third-party tools that misrepresent identity or route third-party traffic against subscription limits, and directs developers building for others to API-key authentication. See [Log in to your Claude account: Authenticating to subscription plans](https://support.claude.com/en/articles/13189465-log-in-to-your-claude-account).

**Consequence:** self-hosting Manifest does not turn a third-party gateway into a native Anthropic application. Manifest cannot accept, refresh, or route Pro/Max OAuth credentials as a product feature. A user can continue using their subscription directly through Claude Code and other Anthropic-supported surfaces.

### Anthropic: Team and Enterprise are per-member products, not a documented credential pool

Anthropic's Commercial Terms govern Team, Enterprise, and API use. They permit API-backed products for a customer's users, but prohibit reselling the services without express approval and make the customer responsible for its account. See [Commercial Terms sections A and D](https://www.anthropic.com/legal/commercial-terms).

Public Team/Enterprise documentation assigns Claude Code to individual seats and has each member authenticate their own account. Seat-based plans track limits and usage-credit spend by organization, group, seat tier, and individual member; current usage-based Enterprise is billed by consumption rather than offering spare subscription quotas to combine. See [Use Claude Code with Team or Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan) and [Manage usage credits for Team and seat-based Enterprise plans](https://support.claude.com/en/articles/12005970-manage-usage-credits-for-team-and-seat-based-enterprise-plans).

Anthropic documents an official self-hosted “Claude apps gateway,” but it fronts the Anthropic API or supported cloud providers with an organization identity provider. It is not documented as a pool of Team/Enterprise seat OAuth tokens. Anthropic's general LLM-gateway guidance likewise describes gateways in front of API/cloud-provider authentication. See [Enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations).

The categorical third-party prohibition names Free, Pro, and Max rather than Team/Enterprise. That omission is **not affirmative permission** to pool commercial seats. The applicable customer agreement, order form, and Anthropic's written interpretation must decide that use.

**Consequence:** until Anthropic approves the exact design in writing, Manifest must treat Team/Enterprise subscription credentials as owner-bound and unavailable to its shared routing plane. For shared workloads, use the Anthropic API, Claude Platform on AWS, Bedrock, Google Cloud's Agent Platform, or Microsoft Foundry.

### OpenAI: individual accounts cannot be shared or used to evade limits

OpenAI's individual Terms of Use apply to personal ChatGPT/Codex access. They prohibit sharing account credentials or making the account available to another person, automatically/programmatically extracting output, and circumventing rate limits, restrictions, protective measures, or safety mitigations. See [OpenAI Terms of Use: Registration and access; Using our Services](https://openai.com/policies/terms-of-use/). OpenAI's separate account-sharing guidance says the account is for the individual who created it, while allowing that person to use it on multiple devices. See [OpenAI Account Sharing Policy](https://help.openai.com/en/articles/10471989-chatgpt-plus-and-chatgpt-pro).

OpenAI does document owner-controlled remote use. Codex can store and refresh a ChatGPT login on a trusted headless host or private CI runner, and a custom model-provider configuration can carry OpenAI authentication through an LLM proxy. Those instructions establish that moving one person's credential to trusted infrastructure is technically supported; they do not authorize sharing it or pooling it with other accounts. See [Codex authentication: headless login, credential storage, and alternative providers](https://developers.openai.com/codex/auth/).

**Consequence:** an owner-bound private proxy may be technically possible, but a hosted or multi-tenant Manifest feature that selects another person's Plus/Pro account—or rotates among accounts to extend capacity—is outside the documented permission. Treat it as blocked unless OpenAI gives written approval.

### OpenAI: Business and Enterprise still require a single end user per seat

Codex usage through ChatGPT Business, Enterprise, or Education is governed by the OpenAI Services Agreement rather than consumer terms. The agreement says account and individual login credentials cannot be shared between users, End User Accounts may be used by only one End User, account access cannot be resold or leased, and customers must not circumvent rate limits or usage limits or configure the services to avoid them. See [OpenAI Services Agreement sections 2 and 3](https://openai.com/policies/services-agreement/).

The agreement expressly permits customer applications for End Users **through the OpenAI API**. It does not give the same grant for pooling ChatGPT/Codex seat credentials. OpenAI documents ChatGPT Enterprise access tokens for trusted scripts, schedulers, and private CI runners when an administrator grants permission, but directs general API calls and programmatic workflows to Platform API keys. See [Codex authentication: enterprise automation](https://developers.openai.com/codex/auth/).

Codex is included across ChatGPT plans, but limits vary by plan and current Codex/agentic usage can draw from a shared workspace or plan credit pool. OpenAI tells users nearing a limit to add authorized credits, upgrade, or wait for reset—not to move work to a different person's seat. See [Using Codex with your ChatGPT plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).

**Consequence:** Business/Enterprise enables centralized administration, not fungible identities. Each Codex seat or access token stays bound to its permitted member/workspace. Shared gateway traffic should use an organization-owned API project unless the customer order form explicitly authorizes another model.

## Allowed, blocked, and unresolved operating modes

| Operating mode | Anthropic | OpenAI | Manifest decision |
| --- | --- | --- | --- |
| Direct interactive native client using the human's own subscription | Supported | Supported | Keep as the reliability baseline |
| Human's subscription used by that same human on another trusted device | Native clients support this | Codex documents trusted headless transfer/device login | Do not ingest into Manifest by default; no cross-user routing |
| Third-party gateway using personal subscription OAuth on behalf of users | Expressly prohibited for Free/Pro/Max | Not granted; account-sharing and limit restrictions apply | Block |
| Rotate among several personal subscriptions after one reaches a limit | Conflicts with ordinary-use/subscriber and anti-routing language | Conflicts with account/usage-limit restrictions | Block, including when one operator owns several accounts |
| Pool Team/Enterprise seats among members | Not publicly authorized | End User Accounts are single-user | Block absent negotiated written authorization |
| Per-user Team/Enterprise automation under the user's own identity | First-party CLI/scripts are documented; gateway interpretation unresolved | Enterprise access tokens are documented for trusted private automation | Owner-bound only; require admin policy and vendor clarification before proxying |
| Organization application using an API key or workload identity | Commercial Terms allow customer products | Services Agreement expressly allows API-backed customer applications | Supported production lane |
| LLM gateway in front of organization API/cloud-provider access | Officially documented | API backend/proxy pattern is documented | Preferred shared routing lane |

## Credential-custody floor

The following controls are the minimum architecture for any credential lane that survives the policy gate. They are derived from the vendors' official credential guidance and the privilege of refresh/access tokens.

### Collection and ownership

- The credential owner performs the provider authorization flow. Manifest never asks for or stores a provider password, emailed login link, MFA response, or browser session cookie.
- Record an immutable binding between the Manifest principal, provider subject/workspace, plan class, permitted purpose, and credential identifier. A request may select only credentials bound to that same authorized principal or an explicitly shared API service account.
- Re-authentication, workspace removal, logout, or user deletion must revoke or delete the stored credential and invalidate active leases.
- Do not import a copied credential from a teammate or accept credentials through support tickets, chat, logs, environment dumps, or repository files.

### Storage and access

- Store secrets in a secrets manager/KMS-backed envelope, not as plaintext application rows. Separate secret material from routing metadata; the main application database should hold only an opaque secret reference and non-sensitive provider metadata.
- Decryption is limited to the dispatch/credential-broker process at request time. The frontend, ordinary support roles, analytics jobs, model router, and logs never receive raw tokens.
- Redact `Authorization`, `X-Api-Key`, cookies, login codes, access tokens, refresh tokens, and credential files at ingestion. Exclude them from telemetry, traces, exception payloads, backups intended for development, and support bundles.
- Apply least privilege, per-tenant isolation, audit every secret read, encrypt backups, and test revocation. Never return a provider credential to a Manifest client after enrollment.

These controls are stricter than local client defaults because a central gateway increases blast radius. Anthropic's API guidance recommends a secrets manager, rotation, scoped workspaces, and workload identity federation for production. See [Anthropic API authentication](https://platform.claude.com/docs/en/manage-claude/authentication). OpenAI says API keys must not be shared or shipped to clients, requests should traverse a secure backend, and production teams should use a key-management service. See [OpenAI API key safety](https://help.openai.com/en/articles/5112595-best-practices-for-api-key).

For subscription credentials, follow at least the first-party client floor. Claude Code stores Linux credentials with mode `0600` and macOS credentials in Keychain; Codex offers OS keyring storage and says a plaintext `auth.json` must be treated like a password. See [Claude Code authentication: Credential management](https://code.claude.com/docs/en/authentication) and [Codex authentication: Credential storage](https://developers.openai.com/codex/auth/).

### Refresh and execution

- Prefer a provider-supported broker over reimplementing private OAuth behavior. For OpenAI, the official Codex app server owns ChatGPT OAuth and refreshes tokens; isolate one broker state directory/security principal per authorized identity. For Anthropic shared production workloads, prefer API keys or workload identity federation rather than subscription refresh tokens.
- Serialize refresh/rotation for each credential, commit replacement material atomically, retain no superseded refresh token, and fail closed on an identity/workspace mismatch.
- Do not spoof a first-party client, change credential audiences/scopes, or forward a credential to an unapproved upstream. Anthropic explicitly prohibits tools that misrepresent their identity.
- Expose health and quota metadata, not secrets. A credential's owner and workspace remain part of every dispatch decision and audit event.

Anthropic offers an inference-only one-year `CLAUDE_CODE_OAUTH_TOKEN` for an eligible subscriber's own CI/scripts, while OpenAI documents automatic ChatGPT token refresh and Enterprise access tokens for trusted automation. These are scoped automation mechanisms, not permission to share or pool subscriptions. See [Claude Code long-lived tokens](https://code.claude.com/docs/en/authentication) and [Codex authentication](https://developers.openai.com/codex/auth/).

### Limits, retries, and failover

- Preserve the provider's account, seat, workspace, organization, model, and reset boundaries. Never treat multiple subscriptions as shards of one quota.
- A subscription limit response is terminal for that credential. Propagate a sanitized limit/reset result or offer an opt-in, separately billed API route owned by the same customer.
- A transient provider error may retry on the **same authorized credential** according to provider guidance. It may fail over to another API credential only when that credential belongs to the same API organization/service identity and the failover does not evade an organization-level restriction.
- Multiple API keys do not manufacture capacity. Anthropic API rate limits apply at the organization level, with optional lower workspace limits, and its API exposes `retry-after`; the gateway must honor those limits. See [Anthropic API rate limits](https://platform.claude.com/docs/en/api/rate-limits).
- Meter and budget per Manifest tenant/principal in addition to vendor limits. Alert on anomalous concurrency, refresh churn, repeated 401s, and attempts to select a credential outside the caller's binding.

## Required vendor questions before reconsidering subscription gateways

Public documentation does not resolve every enterprise case. Obtain written answers that identify the applicable plan and agreement before opening a subscription lane:

### Anthropic

1. May an organization-owned, self-hosted gateway terminate requests for Team or Enterprise subscription OAuth credentials while preserving one seat per named user?
2. May that gateway store/refresh those credentials, or must it use an Anthropic-supported gateway/session-token flow?
3. Is any account selection across seats permitted for service continuity, or must every request remain bound to the initiating member?
4. Does the answer differ for seat-based Enterprise, usage-based Enterprise, Agent SDK use, and interactive Claude Code?
5. Which security, audit, data-processing, and incident-notification terms apply to the credential custodian?

### OpenAI

1. May a self-hosted gateway use ChatGPT/Codex subscription credentials for the same named End User while preserving workspace controls and Compliance API attribution?
2. Does `requires_openai_auth` through a custom proxy authorize only transport proxying, or also server-side custody and refresh by the proxy?
3. May ChatGPT Enterprise access tokens be used by an internal gateway, or only by the member's trusted scripts/CI runners through official Codex clients?
4. Is any failover across Business/Enterprise seats allowed, or must requests remain bound to one End User Account?
5. What contractual route authorizes an organization-wide router if the API is insufficient for a required Codex feature?

Until answered, uncertainty is a deny condition for pooled subscription execution, not an invitation to infer permission.

## Architecture consequences for the Wayfinder map

- Keep Manifest's control plane provider-neutral, but make credential policy a hard capability constraint enforced before model routing.
- Define separate execution classes: `native_direct_subscription`, `owner_bound_enterprise_automation`, and `organization_api`. Do not define a generic `subscription_pool` class.
- The dynamic router may choose among models/providers only inside the caller's authorized execution class. A cache miss, quota exhaustion, latency spike, or provider error cannot broaden credential authority.
- Evaluate native subscription engines only for protocol fidelity and owner-bound execution; their support for loading multiple OAuth files is not evidence that the use is contractually allowed.
- Design the production shared path around API/workload identities and the vendors' official organization/workspace controls. Preserve direct Claude Code and Codex subscription use outside Manifest as the baseline and rollback path.
- Add a release gate requiring current terms/docs review and vendor-approval references for every credential mode. Terms and plan behavior change; this report is a dated snapshot.

## Primary sources

### Anthropic

- [Consumer Terms of Service](https://www.anthropic.com/legal/consumer-terms) — effective October 8, 2025.
- [Commercial Terms of Service](https://www.anthropic.com/legal/commercial-terms) — effective June 17, 2025.
- [Claude Code legal and compliance](https://code.claude.com/docs/en/legal-and-compliance).
- [Claude account authentication and third-party tools](https://support.claude.com/en/articles/13189465-log-in-to-your-claude-account).
- [Claude Code authentication and credential storage](https://code.claude.com/docs/en/authentication).
- [Claude Code enterprise deployment and gateways](https://code.claude.com/docs/en/third-party-integrations).
- [Anthropic API authentication](https://platform.claude.com/docs/en/manage-claude/authentication).
- [Anthropic API rate limits](https://platform.claude.com/docs/en/api/rate-limits).

### OpenAI

- [Terms of Use](https://openai.com/policies/terms-of-use/) — effective January 1, 2026.
- [OpenAI Services Agreement](https://openai.com/policies/services-agreement/) — effective January 1, 2026.
- [OpenAI Account Sharing Policy](https://help.openai.com/en/articles/10471989-chatgpt-plus-and-chatgpt-pro).
- [Using Codex with your ChatGPT plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).
- [Codex authentication, trusted automation, and credential storage](https://developers.openai.com/codex/auth/).
- [Codex app-server authentication behavior](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md#auth-endpoints).
- [OpenAI API key safety](https://help.openai.com/en/articles/5112595-best-practices-for-api-key).
