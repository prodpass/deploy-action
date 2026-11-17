# ProdPass - Record Deployment

Record a deployment in **ProdPass** from GitHub Actions: set the version deployed to an instance (or a component on it), with the run that did it.

```yaml
- uses: hpashok24/prodpass-update-action@v1
  with:
    prodpass_url: ${{ vars.PRODPASS_URL }}
    prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
    instance: order-service.uat2
    version: "1.5.0"
```

- **Fails when nothing was recorded.** ProdPass answers HTTP 200 `"unmatched"` when no instance matches; the step fails instead of passing silently.
- **Safe to retry.** Each attempt gets a unique event ID; network errors and 5xx answers are retried with the same ID, so a deployment is never recorded twice.
- **No dependencies**, nothing downloaded at run time.

## Setup

1. In ProdPass, open **Integrations › Deployment tools**, add **GitHub Actions**, and copy the **URL** and the **token** (shown once).
2. In your GitHub repository, open **Settings › Environments**, create an environment (for example `dev`) and add:

   | Type | Name | Value |
   |---|---|---|
   | Variable | `PRODPASS_URL` | `https://<your-prodpass>/hooks/integrations/<id>` |
   | Secret | `PRODPASS_TOKEN` | the token |

3. Add a workflow, for example `.github/workflows/update-version.yml`:

```yaml
name: Update ProdPass version

on:
  workflow_dispatch:

permissions: {}

jobs:
  update:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: hpashok24/prodpass-update-action@v1
        with:
          prodpass_url: ${{ vars.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          instance: order-service.uat2
          version: "1.5.0"
```

4. Run it from **Actions › Update ProdPass version › Run workflow**. The log shows ProdPass's reply, e.g. `order-service.uat2 1.5.0 recorded as succeeded`.

Keep the version in quotes so values like `1.10` stay text.

## More examples

**At the end of a deploy job**, recording the tag that was deployed and whether the job succeeded:

```yaml
      - uses: hpashok24/prodpass-update-action@v1
        if: always()
        with:
          prodpass_url: ${{ vars.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          instance: order-service.uat2
          version: ${{ github.ref_name }}
          status: ${{ job.status }}
```

**A release deployment**, linked to a ProdPass release. A successful production deployment linked to an approved (go) release can complete it:

```yaml
      - uses: hpashok24/prodpass-update-action@v1
        if: always()
        with:
          prodpass_url: ${{ vars.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          instance: order-service.prod
          version: ${{ github.ref_name }}
          release: REL-00012
          status: ${{ job.status }}
```

This sends:

```json
{
  "instance": "order-service.prod",
  "version": "v1.5.0",
  "status": "succeeded",
  "release": "REL-00012",
  "ref": "abcdef1",
  "url": "https://github.com/acme/order-service/actions/runs/42",
  "finishedAt": "2026-11-17T09:41:12.000Z",
  "eventId": "gh:acme/order-service:42:1:deploy:order-service.prod:succeeded"
}
```

The linked release is available as `steps.<id>.outputs.release`. Leave `release` out and ProdPass matches one by system and version; `release: none` skips linking. Component deployments aren't matched by version, so the action warns when `component` is set without `release`.

**A component of a system**, when the pipeline knows the system and environment rather than the instance code. A new component code adds the component:

```yaml
        with:
          prodpass_url: ${{ vars.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          system: order-service
          environment: UAT2
          component: order-api
          version: "3.1.0"
```

## Inputs

| Input | Required | Description |
|---|---|---|
| `prodpass_url` | ✅ | The URL from ProdPass's GitHub Actions connection. |
| `prodpass_token` | ✅ | The connection's token, from a secret. |
| `instance` | one target | Instance code, or its Pipeline reference when only one instance has it. |
| `system` + `environment` | one target | System and environment codes, instead of `instance`. |
| `component` | | Component code for a system deployed in parts. |
| `version` | ✅ | The version deployed. |
| `status` | | `succeeded` (default), `failed`, `in_progress`, `rolled_back`, or a job status (`success`, `failure`, `cancelled`). Only a successful deployment updates the version. |
| `release` | | Release code or id, e.g. `REL-00012`. Left out, ProdPass matches by system and version (not for components); `none` skips linking. No spaces. |
| `project` | | Project code or id. Left out, the release's project; `none` leaves it empty. |
| `ref` | | Build or change reference. Default: the commit's short SHA. |
| `url` | | Link to the run. Default: this workflow run. |
| `started_at`, `finished_at` | | ISO 8601 times. `finished_at` defaults to now (not for `in_progress`). |
| `event_id` | | Repeat-safe ID. Default: unique per run attempt, job, target and status. |
| `fail_on_unmatched` | | `true` (default) fails the step when no instance matches; `false` only warns. |
| `timeout_seconds` | | Per-request timeout, default `30`. |

## Outputs

| Output | Description |
|---|---|
| `outcome` | `applied` or `unmatched`. |
| `summary` | ProdPass's one-line summary. |
| `event_id` | The event ID sent, to find the call in ProdPass's **Recent calls**. |
| `release` | The release the deployment was linked to (ProdPass's reply, else the one sent). Empty when none. |
| `project` | The project it was linked to. Empty when none. |
| `response` | The raw JSON reply. |

## Notes

- **Instances must exist in ProdPass.** A deployment updates an existing instance; it does not create one. Create instances with **Import and export** or in the ProdPass UI. Components are the exception: a new component code adds the component.
- **Re-runs** are new attempts and are recorded again. Sending the same `event_id` twice is answered `(already recorded)` and changes nothing.
- **Retries:** network errors, timeouts and 5xx answers are retried twice with the same event ID; 4xx answers are not retried.

## Troubleshooting

| Message | Fix |
|---|---|
| `Missing input: prodpass_url` | `PRODPASS_URL` isn't set in the job's environment. Check the job's `environment:` and that the variable exists there. |
| `ProdPass rejected the token (HTTP 401)` | Check the `PRODPASS_TOKEN` secret, that the connection is turned on, and that the token wasn't replaced with **New token**. |
| `ProdPass did not record the deployment: no instance matches "…"` | The instance code or Pipeline reference doesn't exist. Codes are case-sensitive. |
| `ProdPass rejected the deployment (HTTP 400): …` | A field is invalid; the message names it. |
| `Could not reach ProdPass … no answer within 30s` | The runner can't reach ProdPass. For a private ProdPass, use a self-hosted runner. |

## Security

- Keep the token in a secret; the action never prints it.
- The action needs no GitHub token permissions (`permissions: {}`).
- Don't use it in `pull_request_target` workflows, which expose secrets to code from forks.

## Development

```bash
node test/index.test.js
```

Runs the unit tests against a local mock of ProdPass. The **Test** workflow runs them on every push, and can run an end-to-end check against a real ProdPass from **Actions › Test › Run workflow** (needs the `dev` environment above).

To release, tag a version and move the major tag:

```bash
git tag -a v1.0.0 -m "v1.0.0" && git tag -f v1
git push origin v1.0.0 && git push -f origin v1
```

## License

[MIT](LICENSE)
