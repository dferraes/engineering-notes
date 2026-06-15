# Engineering notes

Deep dives on the hard problems I solve — identity, protocols, cloud security, and
platform engineering. No proprietary code; just architecture, decisions, and the parts
that turned out to be hard.

## Write-ups

- **[What I actually look for in a cloud security review](cloud-security-reviews.md)** —
  the IAM, detection, and blast-radius patterns that recur across AWS and GCP reviews,
  and how to sequence findings so the business actually fixes them.
- **[Designing an access-auditing engine across a dozen identity sources](access-auditing-engine.md)** —
  normalizing users, roles, and privileges from AD, three clouds, several databases,
  and a mainframe into one model, then surfacing the accounts that are actually a risk.
- **[Building an ActiveSync + EWS gateway from scratch in Go](activesync-ews-gateway.md)** —
  giving an open-source mail stack a native "Exchange account" experience on iOS,
  macOS, Android and Outlook, including reverse-engineering real clients byte-for-byte.

---

📧 david@ferraes.mx · [LinkedIn](https://linkedin.com/in/davidferraes)
