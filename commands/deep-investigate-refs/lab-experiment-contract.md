# Lab Experiment Contract -- /deep-investigate

## Required fields (fill BEFORE executing)

| Field | Content |
|---|---|
| Experiment ID | EXP-<ticket-id>-<seq> |
| Hypothesis ref | H<n> from claim ledger |
| Objective | Exact hypothesis being tested |
| Parity | Customer conditions reproduced AND conditions omitted (explicit) |
| Versions | Image digests, package versions, source revisions (must match customer where relevant) |
| Baseline | Known working configuration |
| Failure case | Smallest change expected to produce the symptom |
| Prediction | What should happen if hypothesis is correct (written BEFORE execution, NEVER rewritten) |
| Falsification | What result would weaken or refute the hypothesis |
| Correction | Controlled change expected to restore behavior |
| Cloud resources | IDs, tags (investigation:<ticket-id>, expires:<date+7d>), region, account |
| Duration limit | Max time before automatic stop |

## Composition axes

Build the smallest reproduction preserving suspected causal conditions:

- **Product scenario**: logs, APM, monitors, infra, integrations, Terraform, RUM, synthetics, security, profiling, DBM, NDM, CCM
- **Runtime**: Docker/Colima, Kubernetes (kind/minikube), VM, serverless (SAM/CDK), ECS
- **Provider**: AWS, Azure, GCP, on-prem
- **Failure mechanism**: permissions, networking, sampling, parsing, configuration, version regression, race condition, resource limits

## Execution sequence

1. Retrieve keys from Keychain (inject as env vars in subprocess, never print)
2. Provision infrastructure (Colima, cloud resources with tags)
3. Establish healthy baseline, verify it works
4. Introduce the smallest change expected to produce the symptom
5. Compare result against prediction
6. If prediction confirmed: apply correction, verify restoration
7. If prediction failed: update claim ledger, revise hypothesis, create new experiment revision (do NOT rewrite original prediction)
8. For intermittent behavior: define observation window and repetition count, not single test
9. Save all outputs, configs, logs to `lab/EXP-<id>/`
10. Sanitize artifacts before any export or Codex review

## Cleanup contract

After EVERY experiment (success or failure):
1. Discover resources by investigation tag
2. Delete each resource
3. Verify deletion
4. Record cleanup result in `lab/EXP-<id>/cleanup.md`
5. If cleanup fails: record what remains pending, include in manifest.json

Tags do NOT enforce expiry. Cleanup must be actively executed.

## Validity rules

- A single successful retest is weak evidence for intermittent behavior
- Recreating a symptom proves those conditions CAN produce it, not that the customer experienced those conditions
- Every conclusion needs an explicit boundary: what was controlled and what was not
- If hypothesis leads to new hypothesis: new experiment revision, do not retroactively edit predictions
- Parity is a specification, not a label: list what differs from customer and why each difference does or does not matter for this mechanism
