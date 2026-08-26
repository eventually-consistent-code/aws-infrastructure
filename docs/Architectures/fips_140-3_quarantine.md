# FIPS 140-3 Control Tower Migration + Forensics Quarantine

---

## Outcome:

Adopt pre-existing Dev / Collab / Prod accounts into an AWS Control Tower (CT) landing zone **in AWS GovCloud (US)**, enforce a FIPS 140-3 cryptographic baseline everywhere, centralize detection in a Security Tooling account, and make the default incident-response modality "snapshot under FIPS KMS → isolate → copy evidence into a purpose-built Forensics account with an isolated analysis VPC." Every new account — including Forensics — comes out of a code-driven vending pipeline so the baseline is baked in, not bolted on.

**This is a GovCloud plan, and GovCloud changes the architecture, not just the region names.** Three structural facts from the CT-GovCloud documentation drive everything below:

1. **Accounts cannot be created inside GovCloud.** Every GovCloud account is born as a pair — a GovCloud account plus an associated commercial billing account — via the `CreateGovCloudAccount` API, callable *only* from the commercial-partition management account. CT's Account Factory "Create account" feature is removed in GovCloud, and CT cannot create the Audit / Log Archive accounts at landing-zone setup either.
2. **AFT is off the table.** Per AWS's own docs, Account Factory for Terraform cannot be deployed by new AFT customers in GovCloud because CodeStar Connections (third-party VCS integration) isn't available there. Vending moves to a GitLab CI + OpenTofu pipeline we own (Phase 2A) — and losing AFT also deletes the old Terraform-inside-AFT runtime headache: this design is pure OpenTofu end to end.
3. **FIPS is largely the default.** Most GovCloud service endpoints are FIPS-validated out of the box, which shrinks the Phase 3 exception list from "inventory everything" to "verify the stragglers."

| Layer | Choice | Why |
|-------|--------|-----|
| Partition / Regions | AWS GovCloud (US) — home `us-gov-west-1`, secondary `us-gov-east-1` | CUI / ITAR / FedRAMP High boundary; FIPS endpoints default; two-Region pair covers replication + Security Hub aggregation |
| IaC runtime | OpenTofu `1.12.2` (via Terragrunt `1.0.8`) | Licensing preference; Terragrunt v1.x defaults its binary lookup to `tofu` — zero extra config; with AFT gone there's no Terraform anywhere in the stack |
| Orchestration | Terragrunt | DRY across accounts/environments; generates backend + provider wiring |
| Account vending | GitLab CI + OpenTofu pipeline (`aws_organizations_account` with `create_govcloud = true` from the commercial management account) | The only path that exists in GovCloud — AFT can't deploy there (no CodeStar Connections); an MR is still the account-creation workflow |
| VCS + CI | GitLab **self-managed, hosted inside GovCloud** | GitLab CI replaces CodePipeline as the pipeline engine (no CodeConnections dependency), and code + state stay inside the CUI boundary — GitLab.com would put both outside it |

Why this shape: enrollment (not rebuild) preserves the existing workloads, a code-driven vending pipeline makes the baseline reproducible, and evidence-copy-to-Forensics beats analyze-in-place because the compromised account's blast radius never touches your analysis environment…

---

### Phase 1 — Assessment and Preparation of Pre-Existing Accounts

- Inventory every existing account (Dev, Collab, Prod, others): account IDs / root emails / current OU placement / Regions in use / VPCs / AWS Config recorders + delivery channels / CloudTrail trails / KMS keys / security groups / any non-FIPS configuration. **GovCloud addition:** map every GovCloud account to its paired commercial billing account — the pairs share an email address and the commercial side lives in the commercial org; you'll operate credentials in both partitions for the rest of this plan, so record which is which now.
- Start the **FIPS certificate register** here, not at audit time: exact CMVP certificate numbers for every module we'll lean on (KMS HSMs, Nitro crypto, AWS-LC, AL2023/Bottlerocket modules). Hard clock on this — CMVP moves all FIPS 140-2 certificates to the Historical list on **September 21, 2026**; anything still resting on a 140-2 cert stops being defensible then. AWS KMS HSMs already carry a 140-3 Security Level 3 certificate and AWS-LC has 140-3 certs; some OS-module certs are still mid-transition, so verify each one against the active 140-3 list.
- Remediate enrollment blockers:
  - All accounts must live in the same AWS Organizations org that will host the landing zone.
  - Create the `AWSControlTowerExecution` IAM role in every existing account — trusts the management account, AdministratorAccess per AWS docs. Required for enrollment, full stop.
  - **Delete** (not allow-list) pre-existing Config recorders and delivery channels in Regions CT will govern. CT deploys its own baseline recorder; a conflicting recorder fails enrollment. Export existing Config history to S3 first if you need the lineage.
  - Confirm no account is already acting as a Log Archive or Audit account — those are designated once, at landing-zone creation, and never again.
- Target OU structure:
  - **Security OU** — Log Archive + Audit (Security Tooling).
  - **Workloads OU** — Dev, Collab, Prod (child OUs or direct placement).
  - **High-Isolation / Forensics OU** — the new Forensics account, with its own tighter SCP set.
- Plan for auto-enrollment cautiously: the CT-GovCloud docs state Account Factory supports **single-account enrollment only**. Verify whether OU-move auto-enrollment behaves in GovCloud the way it does in commercial before building a runbook around it — assume one-at-a-time until proven otherwise.

### Phase 2 — Deploy Control Tower Landing Zone and Security Accounts

- **Pre-create the Audit and Log Archive accounts — CT will not create them in GovCloud.** From the *commercial* management account, call `CreateGovCloudAccount` twice (Audit, Log Archive); each call creates a GovCloud account plus its commercial billing twin. From the GovCloud management account, invite both GovCloud accounts into the GovCloud org and accept the invitations. Only then does landing-zone setup have what it needs.
- Launch CT from the GovCloud management account in the home Region (`us-gov-west-1`); designate the two pre-created accounts as Log Archive and Audit ("set up CT in an existing organization" flow).
- **At landing-zone creation, supply a customer-managed KMS key** for CloudTrail and Config encryption. This is a setup-time option and retrofitting it later is drift-prone console surgery — do it on day one.
- CT creates and manages the **organization CloudTrail trail** itself — do not create a second org trail; configure the CT-managed one with the KMS key above, delivering to Log Archive.
- Register the target OUs so they're governed.
- Apply mandatory + strongly recommended + elective controls mapping to encryption, data protection, and NIST 800-53 / FedRAMP objectives.
- Account vending moves to Phase 2A — the pipeline has to exist before it can carry the baseline.

### Phase 2A — Account Vending Pipeline (GitLab CI + OpenTofu — the AFT Replacement)

AFT cannot be deployed in GovCloud (no CodeStar Connections for third-party VCS — AWS's docs say so flatly), so we build the vending pipeline ourselves on the tools already in the stack. Smaller surface than AFT, honestly — no Service Catalog, no CodePipeline, no DynamoDB request queue — and the MR-driven workflow survives intact: a reviewed merge request is still how an account gets born.

- **Host GitLab self-managed inside GovCloud** (FIPS-mode AL2023 instances, naturally — it rides the same Phase 3 baseline it delivers). GitLab CI is the pipeline engine; runners live in the management-layer accounts with least-privilege OIDC/role assumption. Code, CI, and state all stay inside the CUI boundary.
- **State backend = GitLab-managed OpenTofu state** (HTTP backend): Terragrunt generates an `http` backend block pointing at the GitLab project's state endpoint (`/api/v4/projects/<id>/terraform/state/<name>`), lock/unlock via POST/DELETE. Auth: `CI_JOB_TOKEN` inside CI, short-lived project access token for local applies — never a long-lived PAT in a dotfile. Self-managed-in-GovCloud means state never leaves the boundary — the exception-list entry GitLab.com would have required simply doesn't exist.
- **Repo layout** (replacing AFT's four):
  - `account-vending` — account requests as HCL; runs in the *commercial* partition management account; `aws_organizations_account` with `create_govcloud = true` creates the GovCloud/commercial pair and captures the GovCloud account ID as an output
  - `account-baseline` — the Phase 3 FIPS 140-3 baseline; applied cross-account into every enrolled account
  - `account-overlays` — per-account deltas (the Forensics VPC lives here)
- **The vending flow, end to end** (two partitions, so two credential contexts — CI stages are explicit about which):
  1. MR to `account-vending` merges → commercial-partition job applies `CreateGovCloudAccount` via OpenTofu → pair exists.
  2. GovCloud-partition job (GovCloud management account): invite the new GovCloud account into the GovCloud org, accept the invitation from the member side (`OrganizationAccountAccessRole`).
  3. Enroll the account into CT (single-account enrollment — see Phase 4) and land it in its target OU.
  4. Baseline job assumes `AWSControlTowerExecution` into the new account and applies `account-baseline`, then any matching `account-overlays` entry. Same job, same code path, for newly vended and newly enrolled pre-existing accounts — one baseline, one delivery mechanism.
- **FIPS posture for the pipeline itself:** `AWS_USE_FIPS_ENDPOINT=true` in every CI job environment; runners on FIPS-mode instances; provider blocks partition-aware (`data.aws_partition` — ARNs are `arn:aws-us-gov:...` here, and hardcoded `arn:aws:` strings are a silent no-match bug in SCPs and trust policies).
- **What we gave up with AFT, for the record:** the managed request-metadata store and AWS-side upgrades. What we got back: pure OpenTofu end to end (the old Terraform-inside-AFT runtime split is gone), one fewer dedicated account, and a pipeline whose every moving part is code we can read.

### Phase 3 — FIPS 140-3 Cryptographic Baseline (All Accounts, Existing and New)

- **KMS:** customer-managed keys only, backed by AWS KMS HSMs (FIPS 140-3 Security Level 3 validated, FIPS-approved mode). Automatic rotation on. Key policies restrict to approved principals.
  - Key policies **cannot** restrict use to FIPS endpoints — no IAM condition key sees which hostname a call arrived on (and in GovCloud the standard KMS endpoint is FIPS-validated anyway). So the model is: enforce client-side (`use_fips_endpoint = true`), **detect** server-side via CloudTrail `tlsDetails` (endpoint hostname + TLS version per API call), alert via a Config/Athena detective control on that field. Scope the alerting to resources tagged `FIPSRequired=true` so the signal stays clean while legacy workloads migrate — preventive where possible, detective where preventive is impossible.
- **Endpoints:** GovCloud does most of this work for us — the standard endpoints for most GovCloud services are FIPS-validated by default; that's a load-bearing reason the workload is in this partition at all. Still force `use_fips_endpoint = true` (AWS CLI config, SDK config, `AWS_USE_FIPS_ENDPOINT=true` env var in Lambda / GitLab CI / SSM automation) so clients resolve the explicit `*-fips` hostname where one exists and the posture is uniform across tooling.
  - *Caveat:* "most" isn't "all." Inventory the endpoints for the services actually in use across `us-gov-west-1` / `us-gov-east-1`, document the handful of stragglers, and note that every AWS endpoint enforces TLS 1.2+ minimum regardless — the exceptions are a compliance-documentation problem, not a plaintext problem.
- **Encryption at rest:** account-level default EBS encryption on, S3 SSE-KMS (or DSSE-KMS where dual-layer is required) — enforced via SCPs + Config rules + the `account-baseline` pipeline. Nitro-based instance families only.
- **Encryption in transit:** TLS 1.2+ with FIPS-approved cipher suites. On ALB/NLB use the FIPS security policies (`ELBSecurityPolicy-TLS13-1-2-FIPS-2023-04` family, backed by AWS-LC FIPS). Security groups / endpoint policies block plaintext protocols.
- **Operating system:** Amazon Linux 2023 or Bottlerocket in FIPS mode (`fips=1` kernel arg + FIPS-enabled crypto policy; validated modules: AL kernel Cryptographic API, OpenSSL FIPS provider, AWS-LC). Validate with `fips-mode-setup --check` / `sysctl crypto.fips_enabled` at boot and report via SSM inventory. Pre-stage memory-capture tooling (AVML or LiME) in the golden AMIs now — a memory dump you can't take in the first minutes of an incident is a memory dump you don't get.
- **SCPs:** deny unencrypted resource creation; Region allowlist is beautifully short in this partition — `us-gov-west-1` and `us-gov-east-1`, deny everything else; write every SCP partition-aware (`arn:aws-us-gov:...` — a hardcoded `arn:aws:` condition silently matches nothing here, which for a Deny statement means the control just isn't there); deny `ec2:ModifySnapshotAttribute` for everyone except the IR role — the Phase 7 evidence-copy mechanism opens a snapshot-sharing path, and this closes it for every other principal; kill-switch SCP that quarantines a whole account under the Forensics OU while carving out the evidence-recovery path (Phase 7).
- **Tag governance:** tags do real work in this design (trigger scoping, ABAC guardrails, evidence selection — Phases 6–8), and tag-based automation inherits the integrity of tag writes. So: AWS Organizations tag policies for hygiene, plus an SCP write-protecting the security tag keys (`Quarantined`, `Environment`, `FIPSRequired`, `IncidentId`) org-wide — deny `iam:TagRole` / `iam:UntagRole` / `TagResource` on those keys (`aws:TagKeys` condition) for everyone except IR and the platform pipeline. A compromised workload that can rewrite `Environment=prod` to `Environment=dev` just downgraded its own approval gate; lock the gate tags as hard as the quarantine tags. Tags are selectors and guardrails here — never the security boundary itself.
- After each existing account enrolls, overlay the remaining FIPS config (Isolation Security Groups, cross-account IR roles, endpoint settings, AMI preferences) via the `account-baseline` pipeline; StackSets only as fallback.

### Phase 4 — Enroll Pre-Existing Dev / Collab / Prod Accounts

GovCloud narrows the menu: the CT-GovCloud docs state Account Factory supports **single-account enrollment only**. So the default here is one account at a time from the CT console (email, display name, IAM Identity Center details, target OU), with existing VPCs untouched. Registering an entire existing OU may still be available for baselining the OU itself — verify its enrollment behavior in this partition before leaning on it, and treat OU-move auto-enrollment the same way: prove it in a sandbox before it goes in a runbook.

- Monitor enrollment; common failures are the missing execution role, Config recorder conflicts, and landing-zone drift.
- Post-enrollment, run the `account-baseline` pipeline to land the full FIPS baseline, Isolation SGs, and IR roles — same code path a freshly vended account gets.
- Sequence enrollments Dev → Collab → Prod, with approval gates on Prod. Single-account enrollment makes the cadence serial anyway; use that to validate the baseline overlay on each Dev account before touching anything louder.

### Phase 5 — Provision the Forensics Account via the Vending Pipeline

- Vend the account through the Phase 2A pipeline into the High-Isolation / Forensics OU — a reviewed MR to `account-vending` creates the GovCloud/commercial pair, the flow invites + enrolls it, and `account-baseline` lands the FIPS baseline, Isolation SGs, and roles automatically. The Forensics-specific build (analysis VPC, one-way KMS, receive-only roles) lives in `account-overlays` so the whole account is reproducible from code.
- Build the **isolated analysis VPC** :
  - No Internet Gateway, no NAT Gateway, no public subnets.
  - Route tables: local routes + interface/gateway endpoints for the minimum FIPS-required service set (SSM, EC2, S3, KMS, CloudWatch Logs).
  - **VPC endpoint policies** locked to the evidence bucket and Forensics KMS keys only — this is what makes "isolated" mean something; an open S3 endpoint is an exfil path.
  - SGs/NACLs: inbound only from approved forensic-operator CIDRs (or none — Session Manager needs no inbound); outbound only to the endpoints.
  - Analysis instances launch from the same FIPS-mode AMIs used everywhere else.
- KMS: Forensics account can decrypt snapshot copies from workload accounts; workload accounts have zero access to Forensics resources. One-way valve.
- Create the **evidence bucket** with Object Lock enabled at creation — it's a creation-time-only flag, there is no retrofit. Compliance mode + a defined retention period for chain of custody; SSE-KMS under the Forensics key.
- Least-privilege cross-account roles: Security Tooling initiates evidence transfer; Forensics is receive-only.

### Phase 6 — Centralized Detection and Continuous Compliance

In the Security Tooling (Audit) account:

- Delegated administrator for GuardDuty, Security Hub, Config, Inspector, Detective, and IAM Access Analyzer. **Macie is not available in GovCloud** — sensitive-data discovery needs a different answer here (Glue/Athena classification jobs, or a third-party FedRAMP-authorized tool) if the requirement is real; don't leave a silent gap where the commercial plan had a service.
- Organization-wide auto-enable for all current and future accounts.
- FIPS endpoints per the Phase 3 posture (mostly default in this partition; verify the stragglers).
- GuardDuty Malware Protection for EC2/EBS — verify current availability of the Malware Protection feature in the GovCloud Regions first (GovCloud feature parity trails commercial; the base detector is there, feature add-ons arrive later) — **and grant the GuardDuty service in your customer-managed EBS key policies**, or scans of CMK-encrypted volumes fail silently into "skipped." Scan *scoping* is tag-native (inclusion/exclusion lists, e.g. a `GuardDutyExcluded` tag): tags decide which instances get scanned, the key grant decides whether CMK-encrypted volumes can be scanned at all — two different levers, don't conflate them.
- Security Hub standards: FSBP, NIST SP 800-53, CIS, cross-Region aggregation to `us-gov-west-1`.
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
   - Tag everything: `IncidentId`, `FindingId`, `Severity`, `Timestamp`, `Quarantined=true`, `FIPSCompliant=true`. The `Quarantined=true` tag is load-bearing: every subsequent IR-role permission carries an `aws:ResourceTag/Quarantined=true` condition, so the playbook physically cannot snapshot, stop, or isolate anything it hasn't first marked in-scope — a blast-radius limiter on our *own* automation, which is exactly the automation an attacker would most like to hijack. (The tag-key protection SCP from Phase 3 is what makes this trustworthy.)
   - Via SSM Session Manager (FIPS/VPC endpoints), collect volatile evidence **before any stop**: processes, network connections, open files, console screenshot, memory dump via pre-staged tooling (AVML or LiME — stage it in the AMI now; you cannot install it mid-incident without stomping evidence). Upload to the evidence bucket (SSE-KMS + Object Lock).
3. **Rapid in-place network isolation (source account)**
   - Swap all SGs on the instance/ENIs for the pre-provisioned Isolation SG: zero inbound (or forensic CIDRs only); outbound limited to the SSM/FIPS VPC endpoints.
   - SG changes do not terminate existing tracked connections — if full containment is required, stop the instance *after* evidence collection, or apply restrictive NACLs (accepting the subnet-wide blast radius).
   - Optionally detach secondary ENIs.
4. **Transfer evidence to Forensics (default modality)**
   - Copy snapshots/AMI to the Forensics account over FIPS endpoints with KMS grants scoped so only Forensics can decrypt. The copy job selects snapshots by `IncidentId` tag rather than threading resource IDs through Step Functions state — fewer moving parts in the state machine, and the tag doubles as chain-of-custody metadata.
   - **Re-encrypt on copy to a Forensics-owned KMS key.** Evidence must not depend on the source account's key surviving — if the account is fully compromised or kill-switched, your chain of custody can't have a dependency back into it.
   - Original instance stays isolated (or stopped) in the source account.
   - Whole-account compromise: move the account into the Forensics OU + apply the kill-switch SCP. The SCP is deny-all **with explicit carve-outs** (condition on the IR role's `aws:PrincipalArn`) for the snapshot/KMS/EC2 actions evidence recovery needs — and remember SCPs never bind the management account or service-linked roles, so the recovery path survives. Keep the carve-out on `aws:PrincipalArn`, not `aws:PrincipalTag`: a principal-tag carve-out is forgeable by anyone in the compromised account holding `iam:TagRole`, and the ARN condition does the same job unforgeably.
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
- **Playbook contents** (deployed via the `account-baseline` pipeline into every enrolled account): protect instance/volumes → snapshot + AMI + tag → volatile evidence via SSM Automation document → Isolation SG → copy to Forensics → optional stop/terminate → notify + update Security Hub.
- **Human-in-the-loop:** Security Hub custom actions let an analyst fire the same playbook on demand; Prod defaults to approval-required.
- **Adapt, don't rebuild:** Automated Security Response on AWS, the GuardDuty automated-response samples, and the `AWSSupport-ContainEC2Instance` SSM runbook — modified to force FIPS endpoints and finish with the Forensics copy.

### Phase 9 — Continuous Validation and Governance

- Config + Security Hub continuously evaluate encryption, public exposure, FIPS-endpoint usage (CloudTrail `tlsDetails` analysis), and OS FIPS-mode indicators via SSM inventory.
- Crypto-module self-test results collected from FIPS-mode instances; alert on failure.
- CloudTrail/GuardDuty watch for non-FIPS API activity.
- Quarterly tabletop: detection → snapshot under FIPS KMS → Isolation SG → copy to Forensics → restore in isolated VPC. If you haven't restored a snapshot in Forensics recently, you don't have a forensics capability — you have a forensics diagram…
- Maintain the **FIPS certificate register**: exact CMVP certificate numbers and update streams for KMS HSMs, Nitro crypto, AWS-LC, AL2023/Bottlerocket modules. Re-verify all of them against the 140-3 active list.

### Phase 10 — Implementation Roadmap

1. Assessment + `AWSControlTowerExecution` roles in all pre-existing accounts; map GovCloud↔commercial account pairs; stand up self-managed GitLab inside GovCloud and create the three pipeline repos (`account-vending`, `account-baseline`, `account-overlays`).
2. Pre-create Audit + Log Archive account pairs via `CreateGovCloudAccount` from the commercial management account; invite into the GovCloud org; deploy the CT landing zone in `us-gov-west-1` with customer-managed KMS at setup; register OUs.
3. Build the vending pipeline: commercial-side `account-vending` job, GovCloud-side invite/enroll job, cross-account baseline job; smoke-test end to end with a throwaway sandbox account pair.
4. Baseline: full FIPS 140-3 `account-baseline` + Isolation SGs.
5. Enroll Dev accounts (single-account enrollment); overlay FIPS config via the baseline pipeline; validate.
6. Vend Forensics account; isolated analysis VPC via `account-overlays`.
7. GuardDuty / Security Hub / Config / Inspector / Detective org-wide; verify Malware Protection availability; CMK key-policy grants for malware scanning; decide the Macie-gap answer.
8. Deploy + test the full snapshot → isolate → Forensics-transfer playbook in Dev, including an actual restore in Forensics.
9. Enroll Collab, then Prod (approval gates on).
10. Operationalize continuous FIPS monitoring, IR runbooks, scheduled Forensics-boundary validation; decommission legacy non-FIPS configuration; publish the final cryptographic-posture doc + certificate register.

---

## Resources

- CMVP FIPS 140-2 → Historical transition: https://csrc.nist.gov/projects/cryptographic-module-validation-program
- AWS FIPS endpoints: https://aws.amazon.com/compliance/fips/
- CT enrollment prerequisites: https://docs.aws.amazon.com/controltower/latest/userguide/enrollment-prerequisites.html
- CT auto-enrollment: https://docs.aws.amazon.com/controltower/latest/userguide/account-auto-enrollment.html
- CT in AWS GovCloud (US) — differences (account creation, BYO Audit/Log Archive, no AFT): https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/govcloud-controltower.html
- `CreateGovCloudAccount` API: https://docs.aws.amazon.com/organizations/latest/APIReference/API_CreateGovCloudAccount.html
- `aws_organizations_account` (`create_govcloud`): https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/organizations_account
- AWS GovCloud (US) User Guide (per-service differences): https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/
- EC2 incident response (AWS Security IR Guide): https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/
- `AWSSupport-ContainEC2Instance`: https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-containec2instance.html
- ELB FIPS security policies: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html
- CloudTrail `tlsDetails`: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html
