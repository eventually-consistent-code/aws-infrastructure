# FIPS 140-3 Control Tower Migration + Forensics Quarantine

**Author(s):** John Reed
**Date:** 2026-08-25 (rev 2 — reworked for the actual stack: OpenTofu + GitLab + AFT-from-scratch)
**Status:** Reviewed draft — plan is sound with corrections (see [Review Verdict](#review-verdict) and [Changes I'd Make](#changes-id-make))

---

## Outcome:

Adopt pre-existing Dev / Collab / Prod accounts into an AWS Control Tower (CT) landing zone, enforce a FIPS 140-3 cryptographic baseline everywhere, centralize detection in a Security Tooling account, and make the default incident-response modality "snapshot under FIPS KMS → isolate → copy evidence into a purpose-built Forensics account with an isolated analysis VPC." Every new account — including Forensics — comes out of Account Factory for Terraform (AFT) so the baseline is baked in, not bolted on.

**Stack this plan is built on** (AFT is *not yet deployed* — standing it up is now Phase 2A, not an assumption):

| Layer | Choice | Why |
|-------|--------|-----|
| IaC runtime | OpenTofu `1.12.2` (via Terragrunt `1.0.8`) | Licensing preference; Terragrunt v1.x defaults its binary lookup to `tofu` — zero extra config |
| Orchestration | Terragrunt | DRY across accounts/environments; generates backend + provider wiring AFT's module deliberately doesn't manage |
| Account vending | AFT module `aws-ia/control_tower_account_factory` `1.20.1` | Only AWS-supported Terraform-native vending path; account requests + customizations as code |
| VCS | GitLab (`vcs_provider = "gitlab"`, or `"gitlabselfmanaged"` if self-hosted) | Team standard; AFT supports it natively via AWS CodeConnections |
| State | S3 + DynamoDB, management account, us-east-1 | No external state services; management account owns all state |

Why this shape: enrollment (not rebuild) preserves the existing workloads, Account Factory makes the baseline reproducible, and evidence-copy-to-Forensics beats analyze-in-place because the compromised account's blast radius never touches your analysis environment…

---

## Review Verdict

The plan is **architecturally sound**. The overall flow — assess → land CT → baseline → enroll → vend Forensics → centralize detection → automate snapshot/isolate/transfer → validate continuously — is the right order and matches AWS-published incident-response and enrollment guidance. It correctly handles the classic enrollment landmines (AWSControlTowerExecution role, AWS Config recorder conflicts, Log Archive / Audit designation being setup-time-only) and the isolation caveats (security-group changes don't kill tracked connections).

But several items as written are **unenforceable, mistimed, or stale**, and one is on a hard clock: FIPS 140-2 certificates move to the CMVP Historical list on **September 21, 2026** — weeks from this writing — so "FIPS-validated" claims need re-verification against active 140-3 certificates *now*, not at audit time. Details in [Changes I'd Make](#changes-id-make).

Two repo-specific corrections up front:

1. This project standardizes on **AFT + OpenTofu + Terragrunt + GitLab**, so everywhere the original plan says "CfCT (Customizations for Control Tower) or StackSets," read "AFT global/account customizations" first, StackSets only where AFT genuinely can't reach (e.g., resources in accounts AFT doesn't manage). Two customization pipelines is one too many.
2. **AFT does not exist yet in this org** — the original plan treated Account Factory as furniture that's already in the room. It isn't. Standing up AFT is its own phase (2A below) with its own prerequisites (dedicated AFT management account, four GitLab repos, a CodeConnections handshake) and its own honest caveat: AFT's internal pipelines run Terraform, not OpenTofu — see the runtime-split note in Phase 2A before anyone promises "OpenTofu everywhere" in an audit.

---

## The Plan (Corrected)

### Phase 1 — Assessment and Preparation of Pre-Existing Accounts

- Inventory every existing account (Dev, Collab, Prod, others): account IDs / root emails / current OU placement / Regions in use / VPCs / AWS Config recorders + delivery channels / CloudTrail trails / KMS keys / security groups / any non-FIPS configuration.
- Remediate enrollment blockers:
  - All accounts must live in the same AWS Organizations org that will host the landing zone.
  - Create the `AWSControlTowerExecution` IAM role in every existing account — trusts the management account, AdministratorAccess per AWS docs. Required for enrollment, full stop.
  - **Delete** (not allow-list) pre-existing Config recorders and delivery channels in Regions CT will govern. CT deploys its own baseline recorder; a conflicting recorder fails enrollment. Export existing Config history to S3 first if you need the lineage.
  - Confirm no account is already acting as a Log Archive or Audit account — those are designated once, at landing-zone creation, and never again.
- Target OU structure:
  - **Security OU** — Log Archive + Audit (Security Tooling).
  - **Workloads OU** — Dev, Collab, Prod (child OUs or direct placement).
  - **High-Isolation / Forensics OU** — the new Forensics account, with its own tighter SCP set.
- Plan for auto-enrollment (verify current landing-zone version supports it — you'll be on 3.3+ anyway on a fresh deploy): moving an account into a registered OU applies baselines and controls automatically.

### Phase 2 — Deploy Control Tower Landing Zone and Security Accounts

- Launch CT from the management account; designate Log Archive and Audit accounts (CT creates them if absent).
- **At landing-zone creation, supply a customer-managed KMS key** for CloudTrail and Config encryption. This is a setup-time option and retrofitting it later is drift-prone console surgery — do it on day one.
- CT creates and manages the **organization CloudTrail trail** itself — do not create a second org trail; configure the CT-managed one with the KMS key above, delivering to Log Archive.
- Register the target OUs so they're governed.
- Apply mandatory + strongly recommended + elective controls mapping to encryption, data protection, and NIST 800-53 / FedRAMP objectives.
- Account Factory configuration moves to Phase 2A — AFT has to exist before it can carry the baseline.

### Phase 2A — Stand Up AFT (GitLab + OpenTofu)

New phase — the original plan assumed Account Factory was already running. Building it:

- **Vend the AFT management account first** via CT's built-in Account Factory (Service Catalog, console — one of the few unavoidable ClickOps moments). AFT requires its own dedicated account; it is not optional and it is not the CT management account.
- **Deploy the AFT module** (`aws-ia/control_tower_account_factory` pinned `1.20.1`) from the management account via OpenTofu 1.12.2 + Terragrunt 1.0.8. Terragrunt generates the backend (S3 `chain-vote-tofu-state-{account_id}` + DynamoDB `chain-vote-tofu-locks`) and the five required provider aliases (`ct_management`, `log_archive`, `audit`, `aft_management`, `tf_backend_secondary_region`) — the AFT module explicitly manages neither.
- **GitLab wiring:** `vcs_provider = "gitlab"` (or `"gitlabselfmanaged"` + instance URL if self-hosted). Four repos, created before apply:
  - `aft-account-request` — account vending requests as HCL; a merge request *is* the account-creation workflow
  - `aft-global-customizations` — the FIPS 140-3 baseline from Phase 3; runs in every vended/enrolled account
  - `aft-account-customizations` — per-account deltas (Forensics VPC lives here)
  - `aft-account-provisioning-customizations` — pre-handoff Step Functions hooks
- **CodeConnections handshake:** AFT reaches GitLab through an AWS CodeConnections connection that must be authorized once, by a human, in the console (OAuth into GitLab). Human gate — put it on the runbook, don't let the pipeline sit "PENDING" for a week wondering why.
- **FIPS posture for AFT itself:** set `AWS_USE_FIPS_ENDPOINT=true` in the customization CodeBuild environments; AFT's request-metadata DynamoDB and artifact buckets ride the baseline SCPs like everything else in the management/AFT accounts.
- **The runtime split — say it plainly:** AFT's CodeBuild pipelines download and run **Terraform** (`terraform_distribution = "oss"`, fetched from releases.hashicorp.com). There is no OpenTofu option (upstream issue #451, open for years). So: every layer *we* drive — CT config, SCPs, the AFT module itself, IR playbooks, Forensics VPC definitions — runs OpenTofu; the execution *inside* AFT's customization pipelines runs Terraform. Two consequences:
  - Keep all customization HCL compatible with both runtimes (Terraform `>= 1.6` feature set, no OpenTofu-only syntax like `.otf` overrides) so nothing forks.
  - **Decision to make and record:** accept the BUSL-licensed Terraform binary running inside the AWS-managed pipeline (running it isn't the licensing concern OpenTofu exists to avoid — but it's a posture inconsistency worth a one-paragraph ADR), or maintain patched buildspecs that swap in `tofu` (unsupported, breaks on every AFT upgrade — I wouldn't). Recommendation: accept + ADR.

### Phase 3 — FIPS 140-3 Cryptographic Baseline (All Accounts, Existing and New)

- **KMS:** customer-managed keys only, backed by AWS KMS HSMs (FIPS 140-3 Security Level 3 validated, FIPS-approved mode). Automatic rotation on. Key policies restrict to approved principals.
  - *Correction:* key policies **cannot** restrict use to FIPS endpoints — no IAM condition key distinguishes `kms-fips.us-east-1` from `kms.us-east-1`; they front the same service. FIPS endpoint usage is enforced client-side (`use_fips_endpoint = true`) and **detected** server-side via CloudTrail `tlsDetails` (endpoint hostname + TLS version per API call). Build a Config/Athena detective control on that field instead of pretending the key policy does it.
- **Endpoints:** force `use_fips_endpoint = true` (AWS CLI config, SDK config, `AWS_USE_FIPS_ENDPOINT=true` env var in Lambda / CodeBuild / SSM automation). All service calls target FIPS endpoints where they exist.
  - *Caveat:* not every service publishes a FIPS endpoint in every Region. Inventory the FIPS endpoint list for each service you actually use in us-east-1, document the exceptions, and note that all AWS endpoints enforce TLS 1.2+ minimum regardless (AWS completed that deprecation in 2023) — the exceptions are a compliance-documentation problem, not a plaintext problem.
- **Encryption at rest:** account-level default EBS encryption on, S3 SSE-KMS (or DSSE-KMS where dual-layer is required) — enforced via SCPs + Config rules + AFT baseline. Nitro-based instance families only.
- **Encryption in transit:** TLS 1.2+ with FIPS-approved cipher suites. On ALB/NLB use the FIPS security policies (`ELBSecurityPolicy-TLS13-1-2-FIPS-2023-04` family, backed by AWS-LC FIPS). Security groups / endpoint policies block plaintext protocols.
- **Operating system:** Amazon Linux 2023 or Bottlerocket in FIPS mode (`fips=1` kernel arg + FIPS-enabled crypto policy; validated modules: AL kernel Cryptographic API, OpenSSL FIPS provider, AWS-LC). Validate with `fips-mode-setup --check` / `sysctl crypto.fips_enabled` at boot and report via SSM inventory.
- **SCPs:** deny unencrypted resource creation; restrict Regions to those with the FIPS endpoint coverage you documented (us-east-1 home region fits); kill-switch SCP that quarantines a whole account under the Forensics OU while carving out the evidence-recovery path (see Phase 7 correction).
- After each existing account enrolls, overlay the remaining FIPS config (Isolation Security Groups, cross-account IR roles, endpoint settings, AMI preferences) via AFT customizations; StackSets only as fallback.

### Phase 4 — Enroll Pre-Existing Dev / Collab / Prod Accounts

Two methods, pick per scale:

**Method A — register an entire existing OU.** Prereqs met for every account under it (execution role present, Config conflicts deleted), then register the OU from the CT console. CT enrolls everything under it, deploys baseline stack sets, applies the OU's controls. Existing VPCs untouched.

**Method B — enroll individually or via auto-enrollment.** Enroll from the CT console (email, display name, IAM Identity Center details, target OU), or — with auto-enrollment on — just move the account into a registered OU via Organizations / `MoveAccount` API.

- Monitor enrollment; common failures are the missing execution role, Config recorder conflicts, and landing-zone drift.
- Post-enrollment, run the AFT overlay to land the full FIPS baseline, Isolation SGs, and IR roles.
- Batch enrollments respecting the current concurrent-account-operations quota (verify it in the console — it has moved over the years; don't hardcode "five"). Start with Dev, then Collab, then Prod with approval gates.

### Phase 5 — Provision the Forensics Account via Account Factory

- Vend the account through AFT under the High-Isolation / Forensics OU — a reviewed merge request to `aft-account-request` is the entire creation workflow. Global customizations land the FIPS baseline, Isolation SGs, and roles automatically; the Forensics-specific build (analysis VPC, one-way KMS, receive-only roles) lives in `aft-account-customizations` so the whole account is reproducible from code.
- Build the **isolated analysis VPC** (calling it "air-gapped" oversells it — it has VPC endpoints; isolated is the honest word, and honest words survive audits):
  - No Internet Gateway, no NAT Gateway, no public subnets.
  - Route tables: local routes + interface/gateway endpoints for the minimum FIPS-required service set (SSM, EC2, S3, KMS, CloudWatch Logs).
  - **VPC endpoint policies** locked to the evidence bucket and Forensics KMS keys only — this is what makes "isolated" mean something; an open S3 endpoint is an exfil path.
  - SGs/NACLs: inbound only from approved forensic-operator CIDRs (or none — Session Manager needs no inbound); outbound only to the endpoints.
  - Analysis instances launch from the same FIPS-mode AMIs used everywhere else.
- KMS: Forensics account can decrypt snapshot copies from workload accounts; workload accounts have zero access to Forensics resources. One-way valve.
- Least-privilege cross-account roles: Security Tooling initiates evidence transfer; Forensics is receive-only.

### Phase 6 — Centralized Detection and Continuous Compliance

In the Security Tooling (Audit) account:

- Delegated administrator for GuardDuty, Security Hub, Config, Inspector, Detective, IAM Access Analyzer, Macie (if used).
- Organization-wide auto-enable for all current and future accounts.
- FIPS endpoints on every service that offers them (per the Phase 3 inventory).
- GuardDuty Malware Protection for EC2/EBS — **and grant the GuardDuty service in your customer-managed EBS key policies**, or scans of CMK-encrypted volumes fail silently into "skipped." Test this; it's the #1 quiet gap in CMK-everywhere shops.
- Security Hub standards: FSBP, NIST SP 800-53, CIS, cross-Region aggregation to us-east-1.
- Config conformance packs evaluating: EBS encryption, public snapshot status, IMDSv2 required, default encryption, and (custom rules + SSM inventory) OS FIPS-mode indicators and FIPS-endpoint usage via CloudTrail `tlsDetails`.
- CloudTrail org trail + Config history land in Log Archive under the FIPS KMS key.

Findings flow GuardDuty/Config → Security Hub → prioritization + automated response.

### Phase 7 — EC2 Snapshot, Isolation, and Quarantine (Default Destination = Forensics)

Trigger: high-severity GuardDuty finding, Security Hub finding, or Config non-compliance on an EC2 instance.

1. **Enrich and protect**
   - EventBridge rule matches on severity ≥ HIGH / resource type EC2 / environment tags.
   - Step Functions assumes the cross-account IR role in the workload account.
   - Enable termination protection AND stop protection on the instance.
   - **Detach from any Auto Scaling group and deregister from load-balancer target groups first** — termination protection does not stop an ASG from replacing-and-terminating the instance, and a half-terminated evidence source is a bad day.
   - Set `DeleteOnTermination = false` on every attached EBS volume.
2. **Create forensic artifacts under FIPS encryption**
   - EBS snapshots of all volumes (customer-managed FIPS KMS key — note: snapshots encrypted with the default `aws/ebs` key **cannot be shared cross-account**, which the CMK requirement already avoids; keep it that way).
   - Snapshots are crash-consistent, not application-consistent — acceptable for forensics; the volatile-evidence step covers what disk misses.
   - Optionally create an AMI.
   - Tag everything: `IncidentId`, `FindingId`, `Severity`, `Timestamp`, `Quarantined=true`, `FIPSCompliant=true`.
   - Via SSM Session Manager (FIPS/VPC endpoints), collect volatile evidence **before any stop**: processes, network connections, open files, console screenshot, memory dump via pre-staged tooling (AVML or LiME — stage it in the AMI now; you cannot install it mid-incident without stomping evidence). Upload to the evidence bucket (SSE-KMS + Object Lock).
3. **Rapid in-place network isolation (source account)**
   - Swap all SGs on the instance/ENIs for the pre-provisioned Isolation SG: zero inbound (or forensic CIDRs only); outbound limited to the SSM/FIPS VPC endpoints.
   - SG changes do not terminate existing tracked connections — if full containment is required, stop the instance *after* evidence collection, or apply restrictive NACLs (accepting the subnet-wide blast radius).
   - Optionally detach secondary ENIs.
4. **Transfer evidence to Forensics (default modality)**
   - Copy snapshots/AMI to the Forensics account over FIPS endpoints with KMS grants scoped so only Forensics can decrypt.
   - **Re-encrypt on copy to a Forensics-owned KMS key.** Evidence must not depend on the source account's key surviving — if the account is fully compromised or kill-switched, your chain of custody can't have a dependency back into it.
   - Original instance stays isolated (or stopped) in the source account.
   - Whole-account compromise: move the account into the Forensics OU + apply the kill-switch SCP. The SCP is deny-all **with explicit carve-outs** (condition on the IR role's `aws:PrincipalArn`) for the snapshot/KMS/EC2 actions evidence recovery needs — and remember SCPs never bind the management account or service-linked roles, so the recovery path survives.
5. **Analysis inside Forensics**
   - Restore snapshots only into the isolated analysis VPC; analysis instances from FIPS-mode AMIs.
   - Malware analysis / memory forensics / timeline reconstruction. No routes, peering, or trust back to Dev/Collab/Prod.
   - FIPS endpoints and validated modules throughout.
6. **Notify and update**
   - SNS (FIPS endpoint) → security-ops channel / ticketing.
   - Update the Security Hub finding: status, notes, links to Forensics-side snapshots.
   - Prod: human-approval gate (Security Hub custom action or Step Functions wait-for-callback) before anything irreversible. Extend the gate to *any* terminate action in *any* environment — terminates are cheap to gate and expensive to regret.

Same pattern extends to other resources (S3 public-access block + object copy, IAM credential revocation, RDS snapshots) with Forensics as the evidence destination.

### Phase 8 — Automated Response Playbooks

- **Trigger layer:** EventBridge rules on GuardDuty findings, Security Hub `Findings – Imported`, Config compliance changes. Filter severity ≥ HIGH / resource type / environment tags.
- **Orchestration:** Step Functions (multi-step visibility beats a Lambda monolith). Execution roles force FIPS endpoints.
- **Cross-account:** Security Tooling assumes least-privilege IR role in the workload account (external ID + condition keys). Forensics has receive-only snapshot permissions.
- **Playbook contents** (deployed via AFT customizations into every enrolled account): protect instance/volumes → snapshot + AMI + tag → volatile evidence via SSM Automation document → Isolation SG → copy to Forensics → optional stop/terminate → notify + update Security Hub.
- **Human-in-the-loop:** Security Hub custom actions let an analyst fire the same playbook on demand; Prod defaults to approval-required.
- **Adapt, don't rebuild:** Automated Security Response on AWS, the GuardDuty automated-response samples, and the `AWSSupport-ContainEC2Instance` SSM runbook — modified to force FIPS endpoints and finish with the Forensics copy.

### Phase 9 — Continuous Validation and Governance

- Config + Security Hub continuously evaluate encryption, public exposure, FIPS-endpoint usage (CloudTrail `tlsDetails` analysis), and OS FIPS-mode indicators via SSM inventory.
- Crypto-module self-test results collected from FIPS-mode instances; alert on failure.
- CloudTrail/GuardDuty watch for non-FIPS API activity.
- Quarterly tabletop: detection → snapshot under FIPS KMS → Isolation SG → copy to Forensics → restore in isolated VPC. If you haven't restored a snapshot in Forensics recently, you don't have a forensics capability — you have a forensics diagram…
- Maintain the **FIPS certificate register**: exact CMVP certificate numbers and update streams for KMS HSMs, Nitro crypto, AWS-LC, AL2023/Bottlerocket modules. Re-verify all of them against the 140-3 active list (see Changes, item 1).

### Phase 10 — Implementation Roadmap

1. Assessment + `AWSControlTowerExecution` roles in all pre-existing accounts; create the four AFT GitLab repos (empty is fine — wiring comes later). (~wk 1–2)
2. Deploy CT landing zone with customer-managed KMS at setup; Security OU accounts; register OUs; auto-enrollment on. (~wk 2–3)
3. Stand up AFT: vend AFT management account via Service Catalog, apply the AFT module (OpenTofu/Terragrunt, `vcs_provider = "gitlab"`), authorize the CodeConnections handshake, smoke-test with a throwaway sandbox account request. (~wk 3–4)
4. AFT baseline: full FIPS 140-3 global customizations + Isolation SGs; record the Terraform-inside-AFT runtime ADR. (~wk 4–6)
5. Enroll Dev accounts; overlay FIPS config via AFT; validate. (~wk 6–7)
6. Vend Forensics account via `aft-account-request` MR; isolated analysis VPC via account customizations. (~wk 7–8)
7. GuardDuty / Security Hub / Config / Inspector org-wide, FIPS endpoints, CMK key-policy grants for malware scanning. (~wk 8–9)
8. Deploy + test the full snapshot → isolate → Forensics-transfer playbook in Dev, including an actual restore in Forensics. (~wk 9–10)
9. Enroll Collab, then Prod (approval gates on). (~wk 10–11)
10. Operationalize continuous FIPS monitoring, IR runbooks, scheduled Forensics-boundary validation; decommission legacy non-FIPS configuration; publish the final cryptographic-posture doc + certificate register. (~wk 11–13)

---

## Changes I'd Make

Ranked. 1–4 are correctness, the rest are hardening.

1. **Re-verify every "FIPS-validated" claim against active 140-3 certificates before September 21, 2026.** CMVP moves all FIPS 140-2 certificates to the Historical list on that date — under a month out. AWS KMS HSMs already carry a 140-3 Security Level 3 certificate; AWS-LC has 140-3 certs; some OS-module certs are still mid-transition. Anything in your register still resting on a 140-2 cert stops being defensible in weeks. Build the certificate register in Phase 1, not Phase 9.
2. **Drop "key policies restrict use to FIPS endpoints" — it can't be done.** No IAM condition key sees which endpoint hostname a call arrived on. Enforce client-side (`use_fips_endpoint`), detect via CloudTrail `tlsDetails`, alert via Config custom rule. Preventive where possible, detective where preventive is impossible.
3. **Re-encrypt evidence to a Forensics-owned key on copy.** As written, Forensics decrypts with grants against the *workload account's* key — a dependency straight back into the compromised blast radius, and one the kill-switch SCP could sever. Copy-with-re-encrypt makes the evidence self-contained.
4. **ASG/ELB detach before isolation.** Termination protection is meaningless to an Auto Scaling group; it will replace-and-terminate your evidence. Detach + deregister is step one of containment, not an afterthought.
5. **Create the evidence bucket with Object Lock enabled from day one** — it's a creation-time-only flag. Compliance mode + a defined retention period for chain of custody.
6. **Grant GuardDuty in every customer-managed EBS key policy** or Malware Protection quietly skips your CMK-encrypted volumes — which, in this design, is all of them.
7. **Supply the customer-managed KMS key at CT landing-zone creation** for CloudTrail/Config, and configure the CT-managed org trail rather than creating a parallel one.
8. **Use AFT customizations, not CfCT/StackSets, as the delivery mechanism** — and stand AFT up first (Phase 2A), since it doesn't exist yet. One customization channel; a second is drift waiting to happen. Record the Terraform-inside-AFT runtime split as an ADR rather than discovering it in an audit.
9. **Rename "air-gapped" to "isolated" and back it with VPC endpoint policies** scoped to the evidence bucket + Forensics keys. The endpoints are the boundary; police them.
10. **Pre-stage memory-capture tooling (AVML/LiME) in the golden AMIs.** A memory dump you can't take in the first minutes is a memory dump you don't get.
11. **Kill-switch SCP needs explicit carve-outs** (IR-role `aws:PrincipalArn` conditions) for the evidence-recovery actions, or the kill switch kills your own investigation.
12. **Document FIPS-endpoint exceptions per service** instead of asserting universal coverage — not every service has a FIPS endpoint in every Region; the audit answer is a documented exception list plus TLS 1.2+ everywhere, not a hopeful blanket claim.
13. **Verify moving parts that drift:** current landing-zone version for auto-enrollment behavior, current concurrent-enrollment quota, current AL2023/Bottlerocket FIPS module certificate status. These change; hardcoding them into the runbook is how runbooks rot.

## Tag-Based Automation: Where It Works (and Where It's a Trap)

Natural follow-up question: can tagging carry some of the enforcement load above — particularly items 2, 6, and 11? Short version: tags are excellent *selectors* for automation and solid ABAC (attribute-based access control) guardrails, but they are not a security boundary unless writes to the tags themselves are locked down. A tag is only as trustworthy as the last principal allowed to write it…

**Item 2 (FIPS endpoints) — no.** Tags attach to principals and resources; the problem is the transport path — which hostname the client called. No IAM condition, tag-based or otherwise, can see the endpoint. Where tags *do* help: scope the detective control. Alert only on non-FIPS calls originating from resources tagged `FIPSRequired=true`, which keeps the signal clean while legacy workloads migrate. Enforcement stays client-side config + CloudTrail `tlsDetails` detection.

**Item 6 (GuardDuty + CMK) — partially.** Two separate mechanisms, don't conflate them:
- The key-policy grant for the GuardDuty service principal is mandatory no matter what — service principals aren't taggable, so there's no ABAC route around it.
- Scan *scoping* is genuinely tag-native: GuardDuty Malware Protection supports tag-based inclusion/exclusion lists (e.g., a `GuardDutyExcluded` tag). Tags decide which instances get scanned; the grant decides whether CMK-encrypted volumes can be scanned at all. Complement, not substitute.

**Item 11 (kill-switch carve-out) — possible, but weaker than `PrincipalArn`.** SCPs support `aws:PrincipalTag` conditions, so a deny-all-except-`PrincipalTag/IRRole=true` carve-out works mechanically. The trap: an attacker inside the quarantined account holding `iam:TagRole` tags their own role and walks out of quarantine. A tag-based carve-out is only safe when paired with a second SCP denying `iam:TagRole` / `iam:UntagRole` / `TagResource` on the security tag keys (`aws:TagKeys` condition) for everyone except IR. At that point you've built two controls to do the job one `aws:PrincipalArn` condition does unforgeably. Keep the ARN condition; treat tags as an optional layer on top, not the load-bearing wall.

**Where tags genuinely earn their keep in this design:**

- **EventBridge trigger scoping** on environment tags — already in Phase 7; severity + resource type + `Environment` tag is the right filter trio.
- **`IncidentId`-driven evidence pipelines** — the Forensics copy job selects snapshots by `IncidentId` tag instead of threading resource IDs through Step Functions state. Fewer moving parts in the state machine, and the tag doubles as chain-of-custody metadata.
- **ABAC guardrail on the IR role itself** (the best addition tagging buys us): playbook step 1 tags the instance `Quarantined=true`; every subsequent IR-role permission carries a `aws:ResourceTag/Quarantined=true` condition. The IR role physically cannot snapshot, stop, or isolate anything that hasn't already been marked in-scope — a blast-radius limiter on our *own* automation, which is exactly the automation an attacker would most like to hijack.
- **Snapshot-sharing lockdown:** SCP denying `ec2:ModifySnapshotAttribute` for everyone except the IR role — the evidence-copy mechanism opens a snapshot-sharing path; this closes it for every other principal.

**Prerequisite for all of it:** AWS Organizations tag policies for hygiene, plus an SCP write-protecting the security tag keys (`Quarantined`, `Environment`, `FIPSRequired`, `IncidentId`) org-wide. Tag-based automation inherits the integrity of tag writes — a compromised workload that can rewrite `Environment=prod` to `Environment=dev` just downgraded its own approval gate. Lock the gate tags as hard as the quarantine tags.

## Open Items / TODO

- [ ] Build the FIPS certificate register (CMVP cert numbers for KMS HSM, AWS-LC, Nitro, AL2023, Bottlerocket) — pre-9/21 deadline
- [ ] Inventory FIPS endpoint availability per service in us-east-1; write the exception list
- [ ] Prototype the CloudTrail `tlsDetails` detective control (Athena query → Config custom rule)
- [ ] Confirm current CT concurrent-account-operation quota + auto-enrollment behavior on the deployed landing-zone version
- [ ] Test GuardDuty Malware Protection against a CMK-encrypted volume end-to-end
- [ ] Stage AVML in the FIPS golden AMI build
- [ ] Write the security-tag-key protection SCP (`Quarantined`, `Environment`, `FIPSRequired`, `IncidentId`) + Organizations tag policies
- [ ] Add `aws:ResourceTag/Quarantined=true` ABAC conditions to the IR role policy
- [ ] Create four AFT GitLab repos + confirm GitLab.com vs self-managed (`vcs_provider` value + instance URL)
- [ ] Write the Terraform-inside-AFT runtime ADR (accept BUSL binary in pipeline vs patched buildspecs)
- [ ] Runbook the CodeConnections GitLab OAuth authorization (human gate)

More to come...

## Resources

- CMVP FIPS 140-2 → Historical transition: https://csrc.nist.gov/projects/cryptographic-module-validation-program
- AWS FIPS endpoints: https://aws.amazon.com/compliance/fips/
- CT enrollment prerequisites: https://docs.aws.amazon.com/controltower/latest/userguide/enrollment-prerequisites.html
- CT auto-enrollment: https://docs.aws.amazon.com/controltower/latest/userguide/account-auto-enrollment.html
- AFT: https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html
- AFT module (pin 1.20.1): https://github.com/aws-ia/terraform-aws-control_tower_account_factory
- AFT alternative VCS (GitLab) config: https://docs.aws.amazon.com/controltower/latest/userguide/aft-alternative-vcs.html
- AFT OpenTofu support (open issue #451): https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/451
- AWS CodeConnections for GitLab: https://docs.aws.amazon.com/dtconsole/latest/userguide/connections-create-gitlab.html
- EC2 incident response (AWS Security IR Guide): https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/
- `AWSSupport-ContainEC2Instance`: https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-containec2instance.html
- ELB FIPS security policies: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html
- CloudTrail `tlsDetails`: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html
