# ProdPass - Record Deployment

Tell **ProdPass** what version you deployed, from GitHub Actions.

## Setup

### 1. Get the URL and token from ProdPass

You need to be a ProdPass admin.

1. Open **Integrations › Deployment tools**.
2. Add a connection and choose **GitHub Actions**.
3. Copy the **URL** and the **token**. The token is shown only once.

Lost the token? Open the connection and click **New token**, then update the GitHub secret. The old token stops working.

### 2. Add them to GitHub as environment secrets

You need admin access to the GitHub repository.

1. In the repository, open **Settings › Environments**.
2. Click **New environment**, name it (e.g. `dev`) and click **Configure environment**. Or click an existing environment.
3. Under **Environment secrets**, click **Add environment secret** and add:

   | Name | Value |
   |---|---|
   | `PRODPASS_URL` | the URL, e.g. `https://<your-prodpass>/hooks/integrations/<id>` |
   | `PRODPASS_TOKEN` | the token |

4. Repeat for each environment you deploy from (e.g. `uat`, `prod`).

The job must name the environment (`environment: dev`), or it can't see these secrets.

### 3. Add the step to your workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      # ... your deploy steps ...

      - uses: prodpass/deploy-action@v1.0
        if: always()
        with:
          prodpass_url: ${{ secrets.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          instance: order-service.uat2
          version: "1.5.0"
          status: ${{ job.status }}
```

`if: always()` and `status: ${{ job.status }}` record failed deploys too.

## Release deployment

Add `release` to link the deployment to a ProdPass release:

```yaml
        with:
          prodpass_url: ${{ secrets.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          instance: order-service.prod
          version: "1.5.0"
          release: REL-00002
          status: ${{ job.status }}
```

- Leave `release` out and ProdPass finds it by version.
- Add `project` only if it's one of the release's projects.

## Component deployment

Use `system` + `environment` + `component` instead of `instance`:

```yaml
        with:
          prodpass_url: ${{ secrets.PRODPASS_URL }}
          prodpass_token: ${{ secrets.PRODPASS_TOKEN }}
          system: order-service
          environment: uat2
          component: order-api
          version: "3.1.0"
          release: REL-00002
```

## Inputs

| Input | Needed | What it is |
|---|---|---|
| `prodpass_url` | Yes | URL from ProdPass |
| `prodpass_token` | Yes | Token from ProdPass |
| `instance` | Yes, or `system` + `environment` | Instance code, e.g. `order-service.uat2` |
| `system`, `environment` | | Use instead of `instance` |
| `component` | | Component code |
| `version` | Yes | Version deployed (keep it in quotes) |
| `status` | | `succeeded` (default), `failed`, `in_progress`, `rolled_back`, or `${{ job.status }}` |
| `release` | | Release code, e.g. `REL-00002` |
| `project` | | Project code |
| `started_at`, `finished_at` | | Times, e.g. `2026-11-17T09:30:00Z` |

Also available: `ref`, `url`, `event_id`, `fail_on_unmatched`, `timeout_seconds`. See `action.yml`.

**Outputs:** `outcome`, `summary`, `release`, `project`, `event_id`, `response`.

## Common errors

| Error | Fix |
|---|---|
| `Missing input: prodpass_url` | Add `PRODPASS_URL` to the environment the job uses (`environment: dev`). |
| `HTTP 401` | Check `PRODPASS_TOKEN`, and that the connection is turned on in ProdPass. |
| `HTTP 404 No project …` / `No release …` | The code doesn't exist in ProdPass. Check **Projects** or **Releases**. |
| `no instance matches` | The instance doesn't exist. Check **System instances**. |
| `no answer within 30s` | GitHub can't reach ProdPass. For a private ProdPass, use a self-hosted runner. |

## Versions

Use a tag: `prodpass/deploy-action@v1.0`. Don't use `@main` in production pipelines.

## Testing

```bash
node test/index.test.js
```

## License

[MIT](LICENSE)
