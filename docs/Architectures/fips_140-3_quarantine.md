# FIPS 140-3 Control Tower Migration + Forensics Quarantine

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

Why this shape: enrollment (not rebuild) preserves the existing workloads, Account Factory makes the baseline reproducible, and evidence-copy-to-Forensics beats analyze-in-place because the compromised account's blast radius never touches your analysis environment…

---

### Phase 1 — Assessment and Preparation of Pre-Existing Accounts

- Inventory every existing account (Dev, Collab, Prod, others): account IDs / root emails / current OU placement / Regions in use / VPCs / AWS Config recorders + delivery channels / CloudTrail trails / KMS keys / security groups / any non-FIPS configuration.
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
- Plan for auto-enrollment (verify current landing-zone version supports it — you'll be on 3.3+ anyway on a fresh deploy): moving an account into a registered OU applies baselines and controls automatically.

### Phase 2 — Deploy Control Tower Landing Zone and Security Accounts

- Launch CT from the management account; designate Log Archive and Audit accounts (CT creates them if absent).
- **At landing-zone creation, supply a customer-managed KMS key** for CloudTrail and Config encryption. This is a setup-time option and retrofitting it later is drift-prone console surgery — do it on day one.
- CT creates and manages the **organization CloudTrail trail** itself — do not create a second org trail; configure the CT-managed one with the KMS key above, delivering to Log Archive.
- Register the target OUs so they're governed.
- Apply mandatory + strongly recommended + elective controls mapping to encryption, data protection, and NIST 800-53 / FedRAMP objectives.
- Account Factory configuration moves to Phase 2A — AFT has to exist before it can carry the baseline.

### Phase 2A — Stand Up AFT (GitLab + OpenTofu)

- **Vend the AFT management account first** via CT's built-in Account Factory (Service Catalog, console — one of the few unavoidable ClickOps moments). AFT requires its own dedicated account; it is not optional and it is not the CT management account.
- **Deploy the AFT module** (`aws-ia/control_tower_account_factory` pinned `1.20.1`) from the management account via OpenTofu 1.12.2 + Terragrunt 1.0.8. Terragrunt generates the backend and the five required provider aliases (`ct_management`, `log_archive`, `audit`, `aft_management`, `tf_backend_secondary_region`) — the AFT module explicitly manages neither.
- **State backend = GitLab-managed OpenTofu state** (HTTP backend) for every layer we drive: Terragrunt generates an `http` backend block pointing at the GitLab project's state endpoint (`/api/v4/projects/<id>/terraform/state/<name>`), lock/unlock via POST/DELETE. Auth: `CI_JOB_TOKEN` inside GitLab CI, short-lived project access token for local applies — never a long-lived PAT in a dotfile. One caveat, stated up front:
  - **AFT keeps its own internal backend regardless.** The module provisions its pipeline state in S3 + DynamoDB inside the AFT management account (with secondary-Region replication — that's what the `tf_backend_secondary_region` provider alias exists for). Not configurable to GitLab. GitLab holds *our* state; AFT holds *its* state.
- **GitLab wiring:** `vcs_provider = "gitlab"` (or `"gitlabselfmanaged"` + instance URL if self-hosted). Four repos, created before apply:
  - `aft-account-request` — account vending requests as HCL; a merge request *is* the account-creation workflow
  - `aft-global-customizations` — the FIPS 140-3 baseline from Phase 3; runs in every vended/enrolled account
  - `aft-account-customizations` — per-account deltas (Forensics VPC lives here)
  - `aft-account-provisioning-customizations` — pre-handoff Step Functions hooks
- **CodeConnections handshake:** AFT reaches GitLab through an AWS CodeConnections connection that must be authorized once in the console (OAuth into GitLab). 
- **FIPS posture for AFT itself:** set `AWS_USE_FIPS_ENDPOINT=true` in the customization CodeBuild environments; AFT's request-metadata DynamoDB and artifact buckets ride the baseline SCPs like everything else in the management/AFT accounts.
- **The runtime split — say it plainly:** AFT's CodeBuild pipelines download and run **Terraform** (`terraform_distribution = "oss"`, fetched from releases.hashicorp.com). There is no OpenTofu option (upstream issue #451, open for years). So: every layer *we* drive — CT config, SCPs, the AFT module itself, IR playbooks, Forensics VPC definitions — runs OpenTofu; the execution *inside* AFT's customization pipelines runs Terraform. Consequence: keep all customization HCL compatible with both runtimes (Terraform `>= 1.6` feature set, no OpenTofu-only syntax like `.otf` overrides) so nothing forks.

### Phase 3 — FIPS 140-3 Cryptographic Baseline (All Accounts, Existing and New)

- **KMS:** customer-managed keys only, backed by AWS KMS HSMs (FIPS 140-3 Security Level 3 validated, FIPS-approved mode). Automatic rotation on. Key policies restrict to approved principals.
  - Key policies **cannot** restrict use to FIPS endpoints — no IAM condition key distinguishes `kms-fips.us-east-1` from `kms.us-east-1`; they front the same service. So the model is: enforce client-side (`use_fips_endpoint = true`), **detect** server-side via CloudTrail `tlsDetails` (endpoint hostname + TLS version per API call), alert via a Config/Athena detective control on that field. Scope the alerting to resources tagged `FIPSRequired=true` so the signal stays clean while legacy workloads migrate — preventive where possible, detective where preventive is impossible.
- **Endpoints:** force `use_fips_endpoint = true` (AWS CLI config, SDK config, `AWS_USE_FIPS_ENDPOINT=true` env var in Lambda / CodeBuild / SSM automation). All service calls target FIPS endpoints where they exist.
  - *Caveat:* not every service publishes a FIPS endpoint in every Region. Inventory the FIPS endpoint list for each service you actually use in us-east-1, document the exceptions, and note that all AWS endpoints enforce TLS 1.2+ minimum regardless (AWS completed that deprecation in 2023) — the exceptions are a compliance-documentation problem, not a plaintext problem.
- **Encryption at rest:** account-level default EBS encryption on, S3 SSE-KMS (or DSSE-KMS where dual-layer is required) — enforced via SCPs + Config rules + AFT baseline. Nitro-based instance families only.
- **Encryption in transit:** TLS 1.2+ with FIPS-approved cipher suites. On ALB/NLB use the FIPS security policies (`ELBSecurityPolicy-TLS13-1-2-FIPS-2023-04` family, backed by AWS-LC FIPS). Security groups / endpoint policies block plaintext protocols.
- **Operating system:** Amazon Linux 2023 or Bottlerocket in FIPS mode (`fips=1` kernel arg + FIPS-enabled crypto policy; validated modules: AL kernel Cryptographic API, OpenSSL FIPS provider, AWS-LC). Validate with `fips-mode-setup --check` / `sysctl crypto.fips_enabled` at boot and report via SSM inventory. Pre-stage memory-capture tooling (AVML or LiME) in the golden AMIs now — a memory dump you can't take in the first minutes of an incident is a memory dump you don't get.
- **SCPs:** deny unencrypted resource creation; restrict Regions to those with the FIPS endpoint coverage you documented (us-east-1 home region fits); deny `ec2:ModifySnapshotAttribute` for everyone except the IR role — the Phase 7 evidence-copy mechanism opens a snapshot-sharing path, and this closes it for every other principal; kill-switch SCP that quarantines a whole account under the Forensics OU while carving out the evidence-recovery path (Phase 7).
- **Tag governance:** tags do real work in this design (trigger scoping, ABAC guardrails, evidence selection — Phases 6–8), and tag-based automation inherits the integrity of tag writes. So: AWS Organizations tag policies for hygiene, plus an SCP write-protecting the security tag keys (`Quarantined`, `Environment`, `FIPSRequired`, `IncidentId`) org-wide — deny `iam:TagRole` / `iam:UntagRole` / `TagResource` on those keys (`aws:TagKeys` condition) for everyone except IR and the platform pipeline. A compromised workload that can rewrite `Environment=prod` to `Environment=dev` just downgraded its own approval gate; lock the gate tags as hard as the quarantine tags. Tags are selectors and guardrails here — never the security boundary itself.
- After each existing account enrolls, overlay the remaining FIPS config (Isolation Security Groups, cross-account IR roles, endpoint settings, AMI preferences) via AFT customizations; StackSets only as fallback.

### Phase 4 — Enroll Pre-Existing Dev / Collab / Prod Accounts

Two methods, pick per scale:

**Method A — register an entire existing OU.** Prereqs met for every account under it (execution role present, Config conflicts deleted), then register the OU from the CT console. CT enrolls everything under it, deploys baseline stack sets, applies the OU's controls. Existing VPCs untouched.

**Method B — enroll individually or via auto-enrollment.** Enroll from the CT console (email, display name, IAM Identity Center details, target OU), or — with auto-enrollment on — just move the account into a registered OU via Organizations / `MoveAccount` API.

- Monitor enrollment; common failures are the missing execution role, Config recorder conflicts, and landing-zone drift.
- Post-enrollment, run the AFT overlay to land the full FIPS baseline, Isolation SGs, and IR roles.
- Batch enrollments respecting the current concurrent-account-operations quota (verify it in the console — it has moved over the years; don't hardcode "five"). Start with Dev, then Collab, then Prod with approval gates.

### Phase 5 — Provision the Forensics Account via Account Factory

- Vend the account through AFT under the High-Isolation / Forensics OU. Global customizations land the FIPS baseline, Isolation SGs, and roles automatically; the Forensics-specific build (analysis VPC, one-way KMS, receive-only roles) lives in `aft-account-customizations` so the whole account is reproducible from code.
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

- Delegated administrator for GuardDuty, Security Hub, Config, Inspector, Detective, IAM Access Analyzer, Macie (if used).
- Organization-wide auto-enable for all current and future accounts.
- FIPS endpoints on every service that offers them (per the Phase 3 inventory).
- GuardDuty Malware Protection for EC2/EBS — **and grant the GuardDuty service in your customer-managed EBS key policies**, or scans of CMK-encrypted volumes fail silently into "skipped." Scan *scoping* is tag-native (inclusion/exclusion lists, e.g. a `GuardDutyExcluded` tag): tags decide which instances get scanned, the key grant decides whether CMK-encrypted volumes can be scanned at all — two different levers, don't conflate them.
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
- **Playbook contents** (deployed via AFT customizations into every enrolled account): protect instance/volumes → snapshot + AMI + tag → volatile evidence via SSM Automation document → Isolation SG → copy to Forensics → optional stop/terminate → notify + update Security Hub.
- **Human-in-the-loop:** Security Hub custom actions let an analyst fire the same playbook on demand; Prod defaults to approval-required.
- **Adapt, don't rebuild:** Automated Security Response on AWS, the GuardDuty automated-response samples, and the `AWSSupport-ContainEC2Instance` SSM runbook — modified to force FIPS endpoints and finish with the Forensics copy.

### Phase 9 — Continuous Validation and Governance

- Config + Security Hub continuously evaluate encryption, public exposure, FIPS-endpoint usage (CloudTrail `tlsDetails` analysis), and OS FIPS-mode indicators via SSM inventory.
- Crypto-module self-test results collected from FIPS-mode instances; alert on failure.
- CloudTrail/GuardDuty watch for non-FIPS API activity.
- Quarterly tabletop: detection → snapshot under FIPS KMS → Isolation SG → copy to Forensics → restore in isolated VPC. If you haven't restored a snapshot in Forensics recently, you don't have a forensics capability — you have a forensics diagram…
- Maintain the **FIPS certificate register**: exact CMVP certificate numbers and update streams for KMS HSMs, Nitro crypto, AWS-LC, AL2023/Bottlerocket modules. Re-verify all of them against the 140-3 active list.

### Phase 10 — Implementation Roadmap

1. Assessment + `AWSControlTowerExecution` roles in all pre-existing accounts; create the four AFT GitLab repos (empty is fine — wiring comes later).
2. Deploy CT landing zone with customer-managed KMS at setup; Security OU accounts; register OUs; auto-enrollment on.
3. Stand up AFT: vend AFT management account via Service Catalog, apply the AFT module (OpenTofu/Terragrunt, `vcs_provider = "gitlab"`), authorize the CodeConnections handshake, smoke-test with a throwaway sandbox account request.
4. AFT baseline: full FIPS 140-3 global customizations + Isolation SGs.
5. Enroll Dev accounts; overlay FIPS config via AFT; validate.
6. Vend Forensics account; isolated analysis VPC via account customizations.
7. GuardDuty / Security Hub / Config / Inspector org-wide, FIPS endpoints, CMK key-policy grants for malware scanning.
8. Deploy + test the full snapshot → isolate → Forensics-transfer playbook in Dev, including an actual restore in Forensics.
9. Enroll Collab, then Prod (approval gates on).
10. Operationalize continuous FIPS monitoring, IR runbooks, scheduled Forensics-boundary validation; decommission legacy non-FIPS configuration; publish the final cryptographic-posture doc + certificate register.

---

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
