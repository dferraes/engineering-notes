# Designing an access-auditing engine across a dozen identity sources

Most "who has access to what" questions in an enterprise are surprisingly hard to
answer. The data lives in a dozen incompatible systems — Active Directory, a handful
of databases, three cloud providers, a mainframe — each with its own model of what a
"user", a "role", and a "privilege" even are. This is a write-up of how I built a
tool that pulls all of that into one normalized model and surfaces the accounts that
are actually a risk.

No proprietary code here — just the design decisions and the parts that turned out to
be hard.

## The shape of the problem

Access auditing splits cleanly into three stages, and keeping them separate is what
makes the whole thing maintainable:

```
  Extract                Normalize + Analyze            Report
  ───────                ────────────────────            ──────
  12 source-specific  →  one identity model       →    PDF / Excel / JSON
  connectors             + risk analyzers               (severity-ranked)
```

The connectors know everything about their source and nothing about analysis. The
analyzers know nothing about where data came from. That boundary is the single most
important design choice in the project — it's why adding a 13th source never touches
the risk logic.

## Connectors: a dozen ways to ask "who are you"

I implemented connectors for Active Directory, Entra ID, Unix/Linux, SQL Server,
MySQL, PostgreSQL, Oracle, AWS, Azure, GCP, eDirectory, and RACF. A few were
genuinely instructive:

- **Active Directory / eDirectory (LDAP)** — paged search is mandatory (directories
  cap results), and AD encodes account state in a `userAccountControl` bitmask and
  timestamps as Windows FILETIME (100-ns ticks since 1601). Half the work is decoding
  representations, not fetching data.
- **Unix/Linux (SSH)** — there's no API; you read `passwd`/`shadow`/`group`/`sudoers`,
  `authorized_keys`, SUID binaries, cron, and `sshd_config` over SSH and reconstruct
  the access model yourself. This connector also runs ~30 platform-specific security
  checks (locked/empty/expired accounts, dangerous SUID binaries, sudo
  misconfigurations, SSH hardening).
- **Cloud (AWS/Azure/GCP)** — three completely different IAM models. AWS is
  users/groups/roles/policies; Azure is RBAC role *assignments* over a scope tree;
  GCP is IAM *bindings* on resources. Normalizing them into one "principal →
  entitlement" shape is most of the effort.
- **RACF (mainframe)** — no live connection at all. You parse the `IRRDBU00` unload —
  a fixed-width flat file, EBCDIC (cp037) — into the same model as everything else.
  Proof that "normalize to one model" pays off: the analyzers treat a 1970s mainframe
  exactly like a 2020s cloud account.

The lesson across all twelve: **the connector's job is translation, not judgment.**
Every source gets mapped to the same internal vocabulary so the analyzers stay
source-agnostic.

## Analyzers: turning entitlements into findings

Once everything is normalized, the risk logic is shared:

- **Orphan accounts** — accounts with no owner, or an owner who's been terminated.
  The trick is *not* flagging legitimate system/service accounts, which look
  ownerless but aren't — so they're explicitly excluded.
- **Dormant accounts** — no activity past a configurable threshold. "Last logon" is
  represented differently in every source (that FILETIME again), so this leans
  entirely on the normalization layer.
- **Over-provisioning** — principals holding more than their peers in the same role,
  a strong signal of privilege creep.
- **Separation of Duties (SoD)** — conflicting entitlements held by one identity (the
  classic "can both create a vendor and approve payments to it"). Rules live in
  external YAML so an auditor can express policy without touching code.
- **Risk scoring** — every finding gets a weighted score so a 5,000-row export
  collapses into a ranked "fix these ten things first" list. An audit that doesn't
  prioritize is an audit nobody acts on.

## Two decisions that mattered

**Ship as a single executable with an encrypted local database.** Auditors run this
against sensitive identity data, often in environments where standing up a server is a
non-starter. So it's a desktop app (PySide6/Qt) that bundles everything into one
binary with an **AES-256 SQLCipher** database — the encryption key derived from the
machine, not stored on disk. No server, no network calls during normal operation,
no audit data leaving the machine. The deployment model *is* a security feature.

**Make extraction restartable and observable.** Pulling from twelve sources, some over
slow SSH or rate-limited cloud APIs, means partial failures are normal. Extraction runs
on a background thread with progress reporting so a connector timing out against one
Oracle box doesn't sink the whole assessment.

## What I'd tell someone building this

- **Normalize ruthlessly, judge centrally.** The payoff isn't visible until the day you
  add a wildly different source (a mainframe, say) and the analysis layer doesn't move.
- **Encode "what's normal" or you'll drown in false positives.** Service accounts that
  look orphaned, break-glass accounts that look dormant — an audit tool is only trusted
  if its findings are trustworthy.
- **Prioritize, don't enumerate.** Anyone can dump every permission in an org. Value is
  telling someone the ten that matter.
- **Let the deployment model do security work.** Single encrypted binary, key derived
  from the host, no network egress — that closes a whole class of "where did the audit
  data go" questions before they're asked.
