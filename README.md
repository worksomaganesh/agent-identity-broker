# Agent Identity & Capability Broker

A planned authorization service for granting AI agents narrow, short-lived permissions to specific tools and resources.

**Agent Identity & Capability Broker** is intended to help replace broad, long-lived credentials with explicit, auditable grants. An agent requests an action; the broker evaluates the identity, policy, resource, and requested operation; the caller receives an appropriately scoped capability or a denial.

> **Project status:** Documentation-first starting point. The repository currently contains this README only. The policy engine, capability issuance, persistence, and tool adapters have not been implemented. The design below is a development target—not a production authorization service.

## The problem

AI agents often operate through tools: repository APIs, ticketing systems, filesystems, deployment systems, databases, or internal services. Giving an agent a broad personal access token or shared service credential can allow much more than the task requires.

Examples of the underlying risks include:

- an agent that only needs to read an issue can also modify repository settings;
- a credential remains valid after a task or session finishes;
- several agents share one identity, making attribution difficult;
- authorization logic is scattered across scripts and tool wrappers;
- revoking one agent's access requires rotating a larger credential;
- logs show that a request happened, but not which policy authorized it.

The broker aims to centralize the decision and make each grant explicit, narrow, time-bounded, and auditable.

## Core concepts

These concepts should remain distinct in both code and documentation:

- **Identity:** which agent, workload, user, or trusted caller is making the request.
- **Authentication:** how the system verifies the caller's claimed identity.
- **Authorization:** whether that identity may perform a specific action on a specific resource under a policy.
- **Capability:** a bounded grant that allows a defined operation or set of operations for a defined resource and period.
- **Audit event:** a record of the request, decision, policy version, and grant or denial, with sensitive material excluded.

A valid identity must not imply broad authorization. A successful authorization decision must not automatically reveal a general-purpose secret to the agent.

## Intended request flow

```text
Agent / trusted tool adapter
             |
             v
    Authenticated request
             |
             v
 Identity and request validation
             |
             v
      Policy evaluation
         /        \
        /          \
      DENY         ALLOW
       |             |
       v             v
 Audit decision   Issue narrow,
                  expiring grant
                       |
                       v
               Resource adapter
                       |
                       v
                Audit outcome
```

The intended design separates the **policy decision point** from the **resource enforcement point**. An adapter that accesses a protected resource must validate the grant and enforce its scope; merely issuing a token is not enough.

## Policy model

Policies should describe the principal, allowed operation, resource boundary, constraints, and approval requirements. The exact schema must be versioned and strictly validated.

Illustrative YAML policy:

```yaml
version: 1
principals:
  - id: coding-agent
    grants:
      - resource: repository:example/project
        actions:
          - repository.read
          - branch.create
          - tests.run
        max_ttl_seconds: 300
    deny:
      - repository.delete
      - branch-protection.modify
```

This is a design example, not a policy accepted by a working implementation. A future policy engine must define conflict precedence, wildcard behavior, defaults, and invalid-policy handling precisely. The safe default should be deny.

## Example API shape

The early API may include endpoints conceptually similar to:

| Operation | Purpose |
| --- | --- |
| Request a capability | Evaluate a principal's request for an operation on a resource |
| Inspect a grant | Return non-secret metadata about a grant and its scope |
| Revoke a grant | Invalidate a grant before its expiry |
| Evaluate a policy | Explain a decision in development or administrative contexts |
| Query audit events | Retrieve authorized, redacted records of decisions |

These are proposed operations, not deployed endpoints. Exact routes and schemas should be chosen only after the policy and enforcement contracts are designed.

## Intended MVP

Start with mock resources, not real credentials or production systems.

1. Define the principal, request, policy, decision, grant, and audit schemas.
2. Implement strict policy parsing and validation.
3. Implement deterministic authorization with default-deny behavior.
4. Issue cryptographically protected, narrowly scoped, expiring capabilities using established libraries.
5. Add revocation and expiry checks, including behavior during restarts.
6. Build a mock tool adapter that validates every request against the grant.
7. Record allow and deny decisions without logging secrets.
8. Add tests for privilege escalation, replay, expiry, revocation, malformed policies, and policy conflicts.
9. Only after the core contract is tested, consider one real integration such as a test repository.

A token or grant must not be treated as secure merely because it is signed. Scope validation, resource-side enforcement, expiry, revocation semantics, key handling, and replay protections all matter.

## Proposed architecture

- **Identity adapter:** establishes a trusted principal from a verified caller context.
- **Request validator:** checks schema, supported action, target, and constraints.
- **Policy engine:** makes a deterministic decision using a validated policy snapshot.
- **Capability issuer:** issues a minimal grant containing audience/resource, allowed actions, issue time, expiry, unique identifier, and key identifier as appropriate.
- **Grant validator:** verifies integrity and enforces scope, expiry, audience, and revocation at the point of use.
- **Resource adapters:** mock protected systems for MVP, with later adapters implementing the same enforcement contract.
- **Audit writer:** records request metadata, decision, policy version, and grant identifier—not bearer-token contents or secrets.
- **Administrative interface:** future, separately authenticated tooling for policy updates and audit review.

Policy changes and cryptographic key rotation require explicit design; they must not be improvised as convenience features.

## Proposed technology choices

These are options to evaluate when implementation begins, not packages already installed in this repository.

- **Language:** Python 3.12+
- **HTTP API:** FastAPI
- **Validation:** Pydantic or equivalent strict models
- **Policy format:** YAML with schema validation
- **Initial persistence:** SQLite; consider PostgreSQL when multi-process or service deployment requires it
- **Capability protection:** established cryptographic/token libraries, selected after defining the threat model
- **Testing:** pytest, with property-oriented and adversarial cases
- **Development environment:** Docker and GitHub Actions where useful

Avoid writing custom cryptographic primitives. Dependency selection should consider maintenance status, security history, supported algorithms, and key-management needs.

## Security model

### Intended protections

The design aims to reduce blast radius by narrowing permissions, bounding grants in time, separating identity from authority, supporting revocation, and recording authorization decisions.

### Threats to test

- an agent requests an action outside its grant;
- a grant is used against a different resource or audience;
- a grant is expired, revoked, malformed, or modified;
- a valid grant is replayed in a context where replay should be rejected;
- a policy is missing, invalid, ambiguous, or changed during a request;
- a caller attempts to impersonate another principal;
- a resource adapter bypasses the broker or fails to validate the capability;
- audit records leak secrets or misattribute an action;
- key rotation makes valid grants unusable or unintentionally keeps invalid grants valid;
- concurrent requests race with revocation or policy changes.

### Non-goals and caveats

- A broker cannot enforce an operation performed outside its controlled adapters.
- A signed capability does not make an untrusted caller trustworthy; the system still needs identity verification.
- Revocation cannot be reliable if resource adapters never check revocation or expiry.
- A local demo with mock resources is not proof of safety for cloud credentials or production infrastructure.
- Human approval, if added, must be bound to the exact action and resource being approved, not a vague session-level permission.

## Testing strategy

Tests should demonstrate least-privilege behavior and failure-closed semantics:

- allow only explicitly granted actions;
- deny unknown actions, unknown resources, missing identity, and invalid policies;
- prevent resource and audience substitution;
- verify expiry at boundary times and across clock-skew assumptions;
- verify revocation before and during concurrent use;
- reject altered or incorrectly signed grants;
- document and test replay behavior;
- prevent over-broad grants from being created accidentally;
- ensure audit events are emitted for both allows and denies;
- ensure logs and error responses never expose bearer tokens or signing keys;
- ensure policy updates have deterministic versioning and conflict behavior.

The project should distinguish unit tests, adapter contract tests, and security regression tests. Tests with mock adapters must not be represented as validation of real provider permissions.

## Roadmap

- [ ] **Milestone 0 — Threat model and contracts:** define principal types, protected resources, capability semantics, and failure behavior.
- [ ] **Milestone 1 — Policy engine:** strict schemas, default-deny evaluation, conflict rules, and explainable decisions.
- [ ] **Milestone 2 — Capability lifecycle:** issuance, validation, expiry, and revocation against mock resources.
- [ ] **Milestone 3 — Auditability:** structured allow/deny records and secret-safe inspection.
- [ ] **Milestone 4 — API and persistence:** authenticated development API and durable state, with migration tests.
- [ ] **Milestone 5 — Adversarial validation:** replay, race, confused-deputy, scope-substitution, and failure-mode tests.
- [ ] **Milestone 6 — One real adapter:** integrate only after the mock enforcement contract is stable.
- [ ] **Milestone 7 — Deployment hardening:** key management, TLS and service identity, operational alerts, backup and recovery, and security review.

## Relationship to an AI Agent Firewall

These projects address different control points:

| System | Primary question |
| --- | --- |
| AI Agent Firewall | Is this attempted action safe and permitted at the runtime boundary? |
| Identity & Capability Broker | Who is requesting the action, and what exact authority has been granted to them? |

They can complement one another. A capability broker should not replace runtime sandboxing or network controls, and a firewall should not be assumed to establish identity or issue least-privilege credentials.

## Contributing

Start with an issue that explains the proposed behavior, threat model, enforcement point, and expected tests. Changes to policy semantics, token claims, revocation, key handling, or fail-open/fail-closed behavior need explicit review.

Never commit API tokens, private keys, production credentials, real identity-provider secrets, or unredacted audit exports. Use synthetic identities and mock protected resources in tests.

## License

A license has not yet been selected for this repository. Until a license file is added, do not assume the code or documentation is available for unrestricted reuse.

## Disclaimer

This repository is at the design stage and is not suitable for protecting real credentials or production systems. Do not connect it to valuable resources until the implementation, adapter enforcement, cryptographic configuration, operational controls, and security testing have been independently reviewed.
