# pharmacyos-updates

This repository publishes the remote update manifest consumed by the PharmacyOS local app.

## Automatic release/update flow

1. Backend and frontend pull requests are merged into their release branches.
2. The frontend release workflow builds the deployable artifact or creates a GitHub Release asset.
3. That release workflow triggers this repository's **Update remote manifest** workflow and passes the release version, release notes, and downloadable artifact URL.
4. This repository updates and commits `manifest.json` on the remote branch.
5. The local app reads `manifest.json` from this repository's raw remote URL. It does **not** need to pull this repository locally.
6. The user clicks **Check for updates** and then **Update Now** in the app.

## Manifest fields

`manifest.json` is maintained automatically and includes the fields the local updater needs:

- `latest_version` — semantic app/frontend version being published.
- `latest_build` — unique build identifier generated as `YYYYMMDDHHMMSS-shortsha`.
- `whats_new` and `release_notes` — human-readable release notes.
- `frontend_artifact_url`, `artifact_url`, and `download_url` — the GitHub-built artifact or release asset URL the updater can download.
- `published_at` — UTC timestamp for when the manifest was updated.

Existing fields such as `channel`, `mandatory`, and `update_size_bytes` are preserved unless the release workflow explicitly sends replacement values.

## Triggering the manifest update from a build workflow

The `repository_dispatch` event is triggered by the frontend/backend release workflow. Nothing inside this update repository fires that event by itself; the source release workflow must dispatch it after the artifact or release asset has been built successfully.

Add this step to the frontend release workflow immediately after the artifact upload or GitHub Release asset creation step:

```yaml
- name: Publish update manifest
  env:
    GH_TOKEN: ${{ secrets.UPDATES_REPO_TOKEN }}
    VERSION: ${{ steps.version.outputs.version }}
    ARTIFACT_URL: ${{ steps.release.outputs.asset_url }}
    RELEASE_NOTES: ${{ steps.notes.outputs.release_notes }}
  run: |
    gh api repos/YOUR_ORG/pharmacyos-updates/dispatches \
      --method POST \
      --field event_type=frontend-release \
      --raw-field client_payload="$(jq -nc \
        --arg latest_version "$VERSION" \
        --arg frontend_artifact_url "$ARTIFACT_URL" \
        --arg release_notes "$RELEASE_NOTES" \
        --arg sha "$GITHUB_SHA" \
        '{latest_version: $latest_version, frontend_artifact_url: $frontend_artifact_url, release_notes: $release_notes, sha: $sha}')"
```

`UPDATES_REPO_TOKEN` must have permission to dispatch events to this repository. The update workflow in this repository uses `contents: write` permission to commit the generated `manifest.json`.

The important wiring is that this dispatch step runs only after the artifact URL is known, so every manifest update points at a downloadable GitHub-built artifact or release asset.

## Manual backfill only

If a release must be repaired, run **Actions → Update remote manifest → Run workflow** and provide:

- `latest_version`
- `artifact_url`
- optional `release_notes`
- optional `channel`
- optional `mandatory`

This is for backfills only. Normal releases should not require manually editing `manifest.json`.

## Local app configuration

Configure the local updater to read the raw remote manifest URL, for example:

```text
https://raw.githubusercontent.com/YOUR_ORG/pharmacyos-updates/main/manifest.json
```

Because the manifest is committed by GitHub Actions in this remote repository, the local machine never needs to pull this repo to see new releases.
