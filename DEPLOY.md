# Staging deployment runbook

## Trigger

A push to `main` runs `.github/workflows/ci.yml` as `test -> build -> deploy`.
`build` and `deploy` run only when `github.event_name == 'push'` and
`github.ref == 'refs/heads/main'`, and only after the previous job succeeds.
Pull requests run `test` only and never publish or deploy.

## Target

- Platform: Render Web Service, deployed from a prebuilt image.
- Image: `ghcr.io/sharingankid/holbertonschool-hbtn-devops-pipeline-lab`
  - `:<commit sha>`: immutable, one per release commit.
  - `:latest`: moving tag, the one the Render service pulls.
- Image visibility: the GHCR package is public. The runtime image contains only
  `src/` and production dependencies; no `.env`, credential, or test file is
  copied into it (see `Dockerfile` and `.dockerignore`).
- The deploy job calls `POST https://api.render.com/v1/services/<id>/deploys`,
  waits until Render reports that deploy as `live` (at most 40 polls, 15 s apart),
  then verifies the endpoints (at most 10 attempts, 10 s apart). Any failure
  makes the job fail.

Configuration, all outside the repository:

| Name | Kind | Where |
|---|---|---|
| `RENDER_API_KEY` | secret | GitHub > Settings > Secrets and variables > Actions |
| `RENDER_SERVICE_ID` | secret | same (`srv-...`) |
| `STAGING_URL` | variable | same, Variables tab, no trailing slash |

## Database

A disposable Render PostgreSQL instance in the same region as the Web Service.
The Web Service environment variable `DATABASE_URL` is set, in the Render
dashboard, to the database's **Internal Database URL**. The application applies
its migrations at startup, so `/items` works on a fresh database.

## Verify

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/health"   # expect 200: process is alive
curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/items"    # expect 200: API can query PostgreSQL
```

`/health` does not touch the database; only `/items` proves the database link.

## Roll back

Redeploy a known-good commit image by its immutable tag. Take the SHA from a
green run in the Actions tab.

Dashboard: Render > service > Settings > Image URL >
`ghcr.io/sharingankid/holbertonschool-hbtn-devops-pipeline-lab:<good sha>` > Save, then
Manual Deploy.

API:

```bash
curl -f -X POST "https://api.render.com/v1/services/$RENDER_SERVICE_ID/deploys" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"imageUrl":"ghcr.io/sharingankid/holbertonschool-hbtn-devops-pipeline-lab:<good sha>"}'
```

Run both verification requests afterwards. Then fix forward, or `git revert`
the bad commit on `main`: the next green run republishes `:latest`. If the
service image URL was pinned to a SHA, set it back to `:latest`.

## Clean up

1. Render: delete the Web Service and the PostgreSQL instance.
2. Render: Account Settings > API Keys > revoke the key used for `RENDER_API_KEY`.
3. GitHub: delete the `RENDER_API_KEY` and `RENDER_SERVICE_ID` secrets and the
   `STAGING_URL` variable.
4. GHCR: set the package back to private, or delete it, per the cleanup policy.
5. Local: `docker logout ghcr.io` and revoke any personal token used for the
   local pull.
