# retail-orders

A demo microservice repo wired to **Beacon**, an enterprise connectivity control plane. Outbound connections are requested by opening a PR that adds an entry to `charts/orders/values.yaml`. Beacon evaluates the PR, posts a signed verdict comment, and — on merge — commits a durable approval record to `main`.

This README is a hands-on walkthrough for a colleague who wants to test the system end-to-end. Architecture deep-dive: [beacon-docs](https://github.com/jkozina/beacon-docs).

---

## Try it (5 minutes)

You'll need:
- `gh` CLI authenticated (`gh auth status` should show `Logged in to github.com`)
- Write access to this repo (ask the maintainer to add you as a collaborator)

```bash
git clone https://github.com/jkozina/retail-orders.git
cd retail-orders
git checkout main && git pull
git checkout -b feat/<your-name>-test
```

Append a new entry under `egress.allow:` in `charts/orders/values.yaml`:

```yaml
    - name: my-test
      host: customers-api.prod.company.internal    # pick from the registry below
      port: 443
      protocol: HTTPS
      justification: <one-line reason — what does this connection enable?>
      ticket: CHG999999
      ttlDays: 30
```

Push and open the PR:

```bash
git commit -am "feat: try beacon"
git push -u origin HEAD
gh pr create --fill
```

Within ~30 seconds the workflow runs, a check appears on the PR, and a Beacon comment is posted.

---

## What happens at each step (and where to look)

| # | What happens | Where to look |
|---|---|---|
| 1 | The push triggers GitHub Actions. The Beacon PDP (`ghcr.io/jkozina/beacon-app:demo`) starts as a workflow service container; at boot it verifies the cryptographically signed OPA policy bundle. | **Actions** tab → "Beacon Connectivity" → most recent run |
| 2 | The Beacon Action parses your `values.yaml`, builds a derived `NetworkIntent` per egress entry, and POSTs to the PDP. The PDP enriches the request with fixture metadata (production: ServiceNow + IPAM + service-catalog), runs the OPA policy, signs the verdict with Ed25519, and returns it. | Run → "Run jkozina/beacon-action@v1" step logs |
| 3 | The action verifies the verdict signature against its embedded public key, then edits a single PR comment in place via an HTML-comment marker. | PR conversation; the comment titled "Beacon Connectivity Verdict — Allow" (or Deny) |
| 4 | The run uploads an artifact called `beacon-evidence` containing four files per intent: the derived intent, the enrichment snapshot, an extraction record, and the signed verdict envelope. | Run page → **Artifacts** → `beacon-evidence` |
| 5a | **Allow** → green check on the PR. The comment shows decisionId, source/destination metadata, controls, matched rules, expiry, and the implementation hash. Merge button is unblocked. | PR comment + check at the bottom of the PR |
| 5b | **Deny** → red check. The comment names the policy rule (e.g. `TTL_EXCEEDS_MAX`) and offers fix-it guidance. Branch protection blocks the merge button. | PR comment + the disabled merge button |
| 6 | **On merge** (allow only): a *second* workflow ("Beacon Record Approval") re-evaluates the PR against current world state and commits `.beacon/approvals/<intent-name>.yaml` to `main`. The commit message references your PR; the durable approval YAML itself contains a fresh Ed25519 signature. | `main` branch → `.beacon/approvals/`; PR timeline shows "referenced this pull request via commit …" |

---

## Destination registry

Pick any of these as the `host` in your egress entry:

| FQDN | Compliance | Classification | Max TTL (days) |
|---|---|---|---|
| `payments-api.prod.company.internal` | pci | restricted | 30 |
| `billing-api.prod.company.internal` | pci | restricted | 30 |
| `fraud-detection-api.prod.company.internal` | pci | restricted | 30 |
| `inventory-api.prod.company.internal` | non-pci | confidential | 90 |
| `customers-api.prod.company.internal` | non-pci | confidential | 90 |
| `notifications-api.prod.company.internal` | non-pci | confidential | 90 |
| `analytics-api.prod.company.internal` | non-pci | confidential | 90 |
| `shipping-api.prod.company.internal` | non-pci | confidential | 90 |
| `search-index-api.prod.company.internal` | non-pci | confidential | 90 |

"Max TTL" is the largest `ttlDays` the policy permits for that destination. Exceeding it triggers `TTL_EXCEEDS_MAX`.

---

## Exercising the deny paths

To see Beacon refuse, try any of these:

| Change to `values.yaml` | What you'll see |
|---|---|
| `host: legacy-rewards-api.prod.company.internal` | `DESTINATION_RETIRED` — the fixture marks the asset as retired in ServiceNow |
| `host: *.anything.com` (any wildcard) | Extraction failed — the action rejects wildcards before calling the PDP |
| `host: anything-not-in-registry.example.com` | `DESTINATION_UNRESOLVED` — Beacon doesn't know that destination |
| `ttlDays: 120` on a `pci/restricted` destination | `TTL_EXCEEDS_MAX` — max for restricted is 30 days |

Each deny produces a structured comment with the rule ID and a suggested fix. The PR's merge button stays disabled.

---

## Verifying the verdict signature yourself

Every verdict carries a `signature` field of the form `beacon-signature:v1:<86-char base64url>` (Ed25519). The public key is published at https://github.com/jkozina/beacon-action/blob/main/keys/beacon-verdict.pub.

```bash
# After your workflow runs, download the artifact
gh run download <run-id> --repo jkozina/retail-orders --name beacon-evidence
cat verdicts/<your-intent>.json | jq '.signature'
```

You can verify the signature with any Ed25519-aware tool. The signed payload is the canonical JSON of the verdict with the `signature` field excluded.

---

## What gets recorded on merge

After you merge an allow PR, look at `main` in two places:

1. **`.beacon/approvals/<intent-name>.yaml`** — the durable, signed approval record. It contains the destination metadata Beacon resolved, the policy bundle revision that produced the verdict, the expiry timestamp, and the signature.
2. **Your PR's timeline** — GitHub auto-links the bot's commit message (`beacon: PR #N — add …`) back to your PR. Click through to find the durable record corresponding to your specific PR.

The post-merge workflow is **diff-aware**: it only writes a new `<name>.yaml` for entries your PR added or modified. Unchanged entries keep their original decisionId from the PR that first approved them. Removed entries' YAMLs get deleted.

---

## Troubleshooting

- **"Required check 'verdict' is expected"** on the PR — the workflow hasn't completed yet, or branch protection requires the check on a commit that didn't trigger it (uncommon). Wait, or push an empty commit to re-trigger.
- **Workflow fails with "Bundle signature verification failed" at PDP startup** — the policy bundle on disk doesn't match its signature. Open an issue; not something you can fix from your PR.
- **PR posts a "Verdict signature failed verification" comment** — the action couldn't verify the PDP's signature with its embedded public key. Usually means the PDP image and the action version drifted; pin or refresh.

---

## Layout

```
charts/orders/values.yaml      # where you add egress entries
.github/workflows/             # the two beacon workflows
  beacon.yml                   # PR-time evaluation
  beacon-record-approval.yml   # post-merge approval recording
.beacon/approvals/             # durable signed approval YAMLs (on main only)
```
