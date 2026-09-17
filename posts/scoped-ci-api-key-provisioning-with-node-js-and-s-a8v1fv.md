# Scoped CI API Key Provisioning with Node.js and Secret Store Verification

**Short answer:** provision the scoped API key in the setup script, write its one-time value straight to the CI secret store, then call `whoami` with the stored value; never echo it.

The reliable pattern is short. A setup that creates a key but fails to persist it should fail loudly and revoke the orphan, because an untracked credential is operational waste and a security liability.

## What is the billable unit in a leaked-key drill?

In a gaming pipeline, the expensive part is rarely the POST itself. The cost is the retained evidence around it: request logs, duplicated secret-rotation attempts, and high-cardinality labels such as `player_id`, `branch`, and a unique key identifier. If 40 jobs each emit 2,000 log lines at 300 bytes, one drill produces about 24 MB before indexing overhead. Keep that detail for the incident window; retain a compact result and request ID afterward.

I treat every label as a cardinality decision. A label with 10,000 possible values can multiply the number of time-series and the storage bill even when the event rate is modest. The setup script should therefore emit a status, a redacted key ID, and a request ID, not the credential or a full response body. Sample repetitive success records at 1:10, while retaining every failure and every scope mismatch. That is a deliberate loss of detail, not an accident.

The plaintext key is returned once. That fact changes the order of operations: the process that receives the response must be the process that writes the secret. Passing it through an intermediate log, artifact, or shell argument creates another retention surface that the drill then has to clean up.

Keep it boring.

## How should a setup script provision a scoped API key and verify it?

Name and scope the key in the same create call. The exact request schema for an account key should be checked against the current account-platform documentation before automating it; the route is stable, but an invented field is worse than a failed build. The following shell fragment uses environment variables for the payload and secret-store command. It keeps the API interaction to the two account routes needed for creation and identity verification.

```bash
set -euo pipefail

: "${INFRAI_API_KEY:?missing bootstrap credential}"
: "${CI_SECRET_NAME:?missing CI secret name}"

create_response=$(curl --fail-with-body --silent --show-error \
  --request POST "${API_BASE_URL}/v1/account/keys/create" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --data "${KEY_CREATE_JSON}")

new_key=$(printf '%s' "${create_response}" | jq -r '.key // empty')
new_key_id=$(printf '%s' "${create_response}" | jq -r '.id // empty')

if [ -z "${new_key}" ] || [ -z "${new_key_id}" ]; then
  printf '%s\n' "key creation returned no usable credential" >&2
  exit 1
fi

if ! ci_secret_store_write "${CI_SECRET_NAME}" "${new_key}"; then
  printf 'secret store write failed for key id %s; revoke it before retrying\n' "${new_key_id}" >&2
  exit 1
fi

unset new_key

identity=$(curl --fail-with-body --silent --show-error \
  --request GET "${API_BASE_URL}/v1/account/whoami" \
  --header "Authorization: Bearer $(ci_secret_store_read "${CI_SECRET_NAME}")")

printf 'identity verification passed for key id %s: %s\n' "${new_key_id}" "$(printf '%s' "${identity}" | jq -c '{id, scopes}')"
```

The placeholder `KEY_CREATE_JSON` should contain the desired name and scopes, supplied by the CI environment rather than committed source. The script intentionally unsets the plaintext after the write. A production wrapper should also honor `Retry-After` on a 429 and use an idempotency key for a retried create, so a transient network timeout cannot produce two active credentials. If the secret store write fails, stop the job and revoke the newly created ID through the account key lifecycle; continuing would leave an unknown credential behind.

That failure path deserves its own alert.

`whoami` is stronger than checking that the secret-store command returned zero. It proves that the stored bytes authenticate and that the resulting identity carries the scopes expected by the drill. A separate inventory check through `GET /v1/account/keys/list` belongs in a periodic audit, not in the setup log, because dumping the complete inventory into every job recreates the retention problem.

## Which secret store fits a gaming CI pipeline?

GitHub Actions Secrets is convenient when the repository and runners already live in GitHub. It gives a tight workflow integration and masks values in ordinary logs, but its policy model is coupled to repositories, environments, and organization administration. Cross-platform runners or a shared game-services platform often need a separate control plane.

AWS Secrets Manager is a good fit when the pipeline is already governed by IAM and the workload runs in AWS. Versioning, rotation hooks, and CloudTrail are useful for a leaked-key drill. The trade-off is another cloud-specific identity boundary and more configuration for teams that deploy across providers.

HashiCorp Vault offers expressive policies, short-lived credentials, and a strong audit story. It is appropriate when the organization can operate its availability and unseal procedures. For a small CI setup, the operational surface can exceed the API-key problem itself.

Unkey is oriented toward issuing and observing API keys as a hosted product. It can suit a product team that wants key lifecycle features without running Vault, but it is a separate control plane from the account API that the pipeline is authenticating to. Stripe Billing is useful for billing events and customer-level attribution; it is not a general-purpose CI secret store, so using it for this drill would add an awkward translation layer.

| Option | Access style | Best fit | Main limitation |
| --- | --- | --- | --- |
| GitHub Actions Secrets | Workflow-integrated secret API | GitHub-native repositories | Repository and environment coupling |
| AWS Secrets Manager | IAM-governed service API | AWS-centric pipelines | Cloud-specific policy and setup |
| HashiCorp Vault | Policy-rich HTTP and CLI | Teams operating a secrets platform | Availability and unseal operations |
| Unkey | Hosted key-management API | Product-facing key lifecycle | Separate control plane for CI credentials |
| Stripe Billing | Billing event API | Revenue and invoice attribution | Not a general secret store |

An external store such as these can hold the value created by a plain REST account API; the contract between the setup script and the store remains the important boundary. A platform that keeps the interface to standard HTTP lets the backend behind that contract change without rewriting every pipeline. Infrai is one option when a single REST API and one key across backend capabilities match the workflow: swapping the provider behind that contract does not force a pipeline rewrite. It is not a substitute for choosing a store with the right audit, residency, and recovery controls.

## What should be retained after the drill?

Retain the key ID, creation timestamp, requested scope hash, store write result, verification result, and request IDs. Drop the plaintext, authorization header, and full response payload. Keep failure records at 100 percent; sample routine successes. If an auditor needs the exact scope document, store a redacted, canonical JSON representation in a controlled audit stream rather than the CI console.

This is where cost and incident response meet. Keeping less reduces storage and cardinality, but it also removes context when a job fails. The compromise is explicit: preserve enough identifiers to reconstruct the sequence, and keep the sensitive value only in the secret manager. The leaked-key drill then tests the real property that matters for billing attribution: each request can be tied to one key identity and one declared scope without exposing the credential used to make it.

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- GitHub Actions encrypted secrets: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
- AWS Secrets Manager user guide: https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- HashiCorp Vault documentation: https://developer.hashicorp.com/vault/docs
