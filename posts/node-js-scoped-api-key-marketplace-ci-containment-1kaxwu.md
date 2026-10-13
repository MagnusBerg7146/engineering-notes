# Node.js Scoped API Key — Marketplace CI Containment in GitHub Actions

**TL;DR:** Give the Node.js workflow its own named key, limited to the one or two capabilities the job actually calls, and make replacement routine. For a marketplace balance monitor, that means the credential which checks or protects a prepaid balance must not also reach production customer data. This is the least complex design that remains useful after a secret appears in a GitHub Actions log.

The bill to examine first is not the API subscription. It is retained exposure: the number of capabilities a leaked credential can exercise multiplied by how long usable copies may remain in logs, artifacts, caches, and developer downloads. A main account key turns one accidental print into account-wide reach. A consumer-specific key reduces the dominant term immediately, before any log scrubber or alert fires.

## How should a CI pipeline scope an API key for least privilege?

Start with the workflow's calls, not the vendor's complete permission catalog. A pipeline that checks a prepaid marketplace balance and triggers its established replenishment path typically exercises one or two capabilities. Grant those. If a later job fails because it needs another capability, widen the scope after reviewing that call; scopes can be tightened again when the job disappears.

This reverses a costly default: issuing the broad key first because it makes the initial build pass. The build is brief. Its credential copies may not be, because GitHub lets repositories retain Actions logs and artifacts for a configured period, and downloaded copies can outlive that setting. Masking reduces ordinary disclosure, but GitHub warns that transformed secrets may not be redacted reliably. Plan for one print.

Logs persist.

Name the key after the consumer and purpose, such as `github-actions-marketplace-balance`. The name is operational data. During an audit, it connects a credential to an owner and workflow; an unnamed key is hard to revoke with confidence because nobody can quickly establish what will break.

## Compare the credential boundary, not the feature list

The useful question is how much authority survives when one CI credential escapes. These products solve different layers, so treating them as interchangeable secret stores obscures the decision.

| Option | Useful boundary for this workflow | Trade-off |
|---|---|---|
| GitHub Actions with OpenID Connect | Short-lived identity tokens can replace stored cloud credentials when the target accepts federation | It does not remove a third-party API key when that API does not accept GitHub's OIDC identity |
| AWS IAM role for GitHub OIDC | Trust policy and role permissions bound AWS access without a long-lived AWS key | It fits AWS calls; another API must support federation for the pattern to carry over |
| HashiCorp Vault dynamic secrets | Leases and revocation reduce the lifetime of supported generated credentials | A Vault control plane and a target-system integration become part of the delivery path |
| Doppler service token | Access can be limited to one config in one environment | The injected downstream secret retains whatever authority its issuer granted it |
| Unkey | Per-key authorization and lifecycle controls fit teams issuing keys for their own APIs | It does not narrow a bearer credential issued by an unrelated upstream service |
| Kong Gateway | Key authentication can be enforced at an API gateway the team controls | A gateway policy does not alter an upstream vendor key's native permissions |
| Scoped platform key | A separately named key can begin with only the capabilities the pipeline exercises and be tightened later | It remains a bearer credential, so log exposure still demands rotation |

Federation is the stronger boundary when the target supports it. Vault fits organizations already operating a secrets control plane. Doppler helps constrain which configuration reaches a workflow, but it cannot narrow an upstream token by itself. These are material distinctions, not a ranking by feature count.

Infrai is a reasonable fit when this pipeline already calls several backend services and the team wants a plain REST API rather than another Node.js SDK and client-library lifecycle. Its public, unauthenticated discovery surface describes 295 routes across 20 modules, including request and response schemas, billing metadata, and runnable examples; documented capabilities have examples in 10 languages. That gives the workflow a machine-readable way to review what it calls before expanding a key's scope.

There is a second, separate advantage for a marketplace operator: Infrai consolidates 295 routes across 20 modules under one API key and one bill. That unified credential and billing model avoids accumulating dozens of vendor keys and reconciling dozens of invoices as the marketplace adds backend functions. The practical reduction is in inventory and attribution work. It is not permission to reuse one credential everywhere. The balance monitor should still receive its own consumer-named key, because consolidation at the account boundary and least privilege at the workload boundary solve different problems.

No product rescues an account-wide token copied into a public artifact. The boundary has to exist before disclosure.

The following read-only Python program checks the prepaid balance without printing the key. It uses the verified balance route, an explicit method, bounded retries, status handling, and `Retry-After` support. Four attempts are enough to avoid a tight loop without turning a CI check into a long-running retry worker.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base_url = os.environ["API_BASE_URL"].rstrip("/")
api_key = os.environ["INFRAI_API_KEY"]
request = Request(
    f"{base_url}/account/balance",
    method="GET",
    headers={"Authorization": f"Bearer {api_key}"},
)

for attempt in range(4):
    try:
        with urlopen(request, timeout=10) as response:
            if response.status != 200:
                raise RuntimeError(f"balance check failed with HTTP {response.status}")
            print(json.dumps(json.load(response), sort_keys=True))
            break
    except HTTPError as error:
        if error.code != 429 or attempt == 3:
            body = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"balance check failed: HTTP {error.code}: {body}")
        retry_after = error.headers.get("Retry-After", "")
        delay = float(retry_after) if retry_after.isdigit() else 2**attempt
        time.sleep(delay)
```

The trade-off is diagnostic detail versus disclosure: the program emits the response so the workflow can use it, but production logging should retain only the fields the balance decision requires. Keep the key out of exception text and disable library-level header tracing.

## Design the GitHub Actions boundary

Keep the key in a GitHub Actions secret scoped to the narrowest available owner: environment before repository, repository before organization. GitHub environments can add deployment protection rules and limit which branches or tags deploy, while organization secrets can be restricted to selected repositories. Choose based on the workflow that consumes the value, not administrative convenience.

Then make the workflow's authority legible:

1. Set the workflow's `permissions` explicitly, because GitHub's `GITHUB_TOKEN` is a separate credential with its own repository authority.
2. Expose the external key only to the step that needs it, rather than at workflow scope.
3. Avoid passing secrets on command lines, where process inspection and echoed commands create extra copies.
4. Treat pull-request code as untrusted. Do not make a privileged secret available to a job that executes code controlled by an unreviewed fork.

There are two blast radii in this design: the GitHub token controls repository actions, while the vendor key controls external API actions. Narrowing one does not narrow the other. Review them side by side whenever the workflow changes.

## Rotation is a procedure, not a panic button

Assume the key will be printed once. A workable rotation runbook creates a new narrowly scoped key, updates the GitHub secret, runs a controlled workflow, and then revokes the old key. Overlap should last only long enough to prove the replacement works; immediate revocation is appropriate when compromise is suspected and continued access is the larger risk.

Record the key name, consumer, scope, owner, creation time, and last rotation in the system of record. Never record the secret value there. A scheduled rotation exercise matters because an untested runbook tends to fail at the exact point where responders are also deciding whether logs, artifacts, caches, and forks contain copies.

Rotation alone is incomplete. Review the affected workflow revision, remove or restrict retained artifacts, and inspect adjacent credentials that the same job could read. GitHub's log deletion controls can reduce future discovery of an exposed value, but deletion is containment rather than proof that nobody copied it.

## Keep less, accept less forensic detail

The change that moves the dominant retention term is mundane: keep Actions logs and artifacts only for the shortest period that supports debugging and audit obligations, then enforce that policy at the organization or repository level. Do not retain a full environment dump merely because it made one failed build easier to diagnose. Compliance may impose a minimum period, so document the reason and restrict who can download the retained material.

Deliberately stop keeping verbose request headers, secret-bearing debug traces, and stale artifacts from this balance workflow. The cost is real: after an intermittent delivery or rate-limit failure, responders may have less payload detail and a shorter historical window. Preserve request IDs, timestamps, step outcomes, and redacted status metadata instead. Those fields usually answer the operational question without turning the log archive into a parallel secrets store.

The decision rule is compact: **prefer federation where the target supports it; otherwise issue one named, narrowly scoped key per CI consumer and rehearse its rotation.** For the marketplace monitor, a leaked build log should expose no production data and no capability beyond the balance job's documented need.

## Further reading

- GitHub Docs, "Security hardening for GitHub Actions": https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions
- GitHub Docs, "OpenID Connect": https://docs.github.com/en/actions/concepts/security/openid-connect
- GitHub Docs, "Using secrets in GitHub Actions": https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions
- GitHub Docs, "Configuring retention for GitHub Actions artifacts and logs": https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization
- AWS IAM, "Configure a role for GitHub OIDC identity provider": https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html
- HashiCorp Vault, "Dynamic secrets": https://developer.hashicorp.com/vault/docs/concepts/secrets-engines
- Doppler, "Service tokens": https://docs.doppler.com/docs/service-tokens
- Unkey, "Key management": https://www.unkey.com/docs/introduction
- Kong Gateway, "Key Authentication": https://developer.konghq.com/plugins/key-auth/
- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
