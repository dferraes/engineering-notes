# What I actually look for in a cloud security review

I've run a lot of architecture and security reviews across AWS and GCP — Well-Architected
reviews, IAM audits, and the kind of "is this safe to ship?" pass you do before a workload
goes to production. The same handful of problems show up again and again, in both clouds,
regardless of team size. This is the checklist I've built from that, and the reasoning
behind it.

No client specifics here — just the patterns and how I think about them.

## Start with blast radius, not with a control list

The first question on any review isn't "is encryption on?" It's **"if this one credential
or component is compromised, how far does the damage reach?"** Every other finding is a
way of shrinking that radius. Framing the review around blast radius keeps it from
becoming a checkbox exercise and forces prioritization: a wildcard admin role is a bigger
deal than a missing log retention setting, even though both are "findings".

## IAM is where the real risk lives

In both clouds, identity is the perimeter. The recurring problems:

- **Wildcard permissions.** `"Action": "*"` on AWS, or `roles/owner` / `roles/editor`
  handed out broadly on GCP. Almost always "we'll tighten it later" that never happened.
  The fix is boring and effective: scope to the specific actions and resources actually
  used (CloudTrail / IAM Access Analyzer on AWS, IAM Recommender / Policy Analyzer on GCP
  will tell you what's actually used vs granted).
- **Humans with standing access.** Long-lived IAM users with access keys (AWS) or user
  accounts holding project-level roles (GCP) instead of federated SSO + short-lived
  credentials. The target is **zero long-lived human credentials** — SSO/identity
  federation, roles assumed on demand, keys that expire.
- **Over-trusting service identities.** Service accounts / instance roles with far more
  than the workload needs, and cross-account / cross-project trust that's wider than
  intended. Non-human identities now outnumber humans and rarely get reviewed — they're
  the quiet majority of privilege creep.
- **The root/owner account.** AWS root with no MFA and active keys; GCP org with no
  break-glass discipline. Root should be locked down, MFA'd, and effectively never used.

If I only had an hour, I'd spend it here.

## Detection: you can't respond to what you can't see

A workload with great preventive controls and no detection is one missed config away from
a silent breach. Baseline I expect on:

- **AWS** — CloudTrail on (all regions, log file validation), GuardDuty enabled, Security
  Hub aggregating findings, AWS Config recording resource state and flagging drift.
- **GCP** — Security Command Center on, audit logs (admin + data access) flowing, and
  findings actually routed somewhere a human sees them.

The common gap isn't that these are off — it's that they're **on but nobody's watching
the output.** A finding that lands in a console nobody opens is not detection.

## The boring controls that still matter

- **Encryption** — on by default almost everywhere now, so the real question is *key
  management*: customer-managed keys (KMS/Cloud KMS) where the data sensitivity justifies
  it, with rotation and access to the key itself controlled.
- **Public exposure** — public S3 buckets / GCS buckets, security groups open to
  `0.0.0.0/0`, databases on public subnets. Block Public Access and org policy
  constraints exist precisely to make this hard to do by accident — turn them on at the
  org level so it's the default, not a per-resource decision.
- **Network segmentation** — flat networks where everything can reach everything. Private
  subnets, VPC Service Controls (GCP) / endpoint policies (AWS), and egress control so a
  compromised workload can't freely call out.

## Security and cost are the same conversation

Reviews that ignore cost get ignored by the business. Some of the highest-impact findings
sit at the intersection: idle and forgotten resources are both spend *and* unmonitored
attack surface; oversized, over-permissioned infrastructure costs more and fails harder.
I've found that leading with "here's what this saves *and* secures" gets remediation
prioritized far faster than a pure-security framing. Right-sizing and decommissioning
dead resources is often the cheapest security win on the table.

## How I sequence the findings

A review that returns 60 findings with no order is noise. I rank by blast radius and
likelihood:

1. **Stop the bleeding** — anything internet-exposed + over-privileged. Public + admin =
   fix today.
2. **Shrink identity** — wildcard policies, standing human credentials, root hardening.
3. **Turn on the lights** — detection coverage and routing findings to someone.
4. **Defense in depth** — CMK, segmentation, egress, drift detection.
5. **Hygiene** — log retention, tagging, cost cleanup (which doubles as attack-surface
   cleanup).

## What I'd tell someone running their first review

- **Lead with blast radius.** It turns a flat checklist into a priority order the business
  understands.
- **Read what's actually used, not what's granted.** Access Analyzer / IAM Recommender turn
  "tighten IAM someday" into a concrete diff.
- **Set safe defaults at the org level.** Block-public-access and org policy constraints
  beat catching mistakes one resource at a time.
- **Frame security in business terms.** "Saves money and reduces attack surface" gets
  remediated; "violates a control" gets a ticket nobody picks up.
