# Smart Intersection — local fixes

This repo does **not** mirror the Smart Intersection sample app itself — install that from the
official source as usual. This repo holds a single patch fixing two issues found while running
it, so you can re-apply the fix after a fresh install on a new machine.

## What the patch fixes

`0001-smart-intersection-fix.patch` touches 4 files under
`metro-ai-suite/metro-vision-ai-app-recipe/`:

- **`compose-scenescape.yml`** — adds a one-shot `dlstreamer-pipeline-starter` container, and
  changes `node-red` to build from a local Dockerfile instead of pulling the stock image.
- **`smart-intersection/src/dlstreamer-pipeline-server/config.json`** — sets `auto_start: false`
  on all 4 camera pipelines (now started by the new container instead).
- **`smart-intersection/src/dlstreamer-pipeline-server/start_pipelines.py`** *(new file)* — starts
  the 4 pipelines one at a time over the REST API.
- **`smart-intersection/src/node-red/Dockerfile`** *(new file)* — builds node-red with
  `node-red-contrib-influxdb` baked in at image build time.

**Why:** `dlstreamer-pipeline-server`'s built-in `auto_start` launches all 4 camera pipelines
concurrently the instant it finishes booting. This reliably crashes every one of them — the
gvapython `datapublisher` callback in each pipeline races to open an MQTT connection at the same
instant and fails for all of them. Starting them sequentially from an external one-shot container
avoids the race and survives restarts/reboots, instead of requiring a manual REST call every time.

## How to use

1. Install the Smart Intersection sample app from the official source, following the
   [official Get Started guide](https://github.com/open-edge-platform/edge-ai-suites/blob/main/metro-ai-suite/metro-vision-ai-app-recipe/smart-intersection/docs/user-guide/get-started.md):

   ```bash
   git clone --filter=blob:none --sparse --branch main https://github.com/open-edge-platform/edge-ai-suites.git
   cd edge-ai-suites
   git sparse-checkout set metro-ai-suite
   cd metro-ai-suite/metro-vision-ai-app-recipe/
   ./install.sh smart-intersection
   ```

2. Get this patch and apply it **from the `edge-ai-suites` repo root** (one level above
   `metro-ai-suite/`, where you ran `git sparse-checkout set` in step 1):

   ```bash
   cd edge-ai-suites
   curl -O https://raw.githubusercontent.com/collins8471/smart_intersection/main/0001-smart-intersection-fix.patch
   git am 0001-smart-intersection-fix.patch
   ```

   If `git am` complains the tree is dirty or the patch doesn't apply cleanly (e.g. upstream
   has since changed those files), fall back to:

   ```bash
   git apply --3way 0001-smart-intersection-fix.patch
   ```

3. Rebuild the node-red image (now built locally instead of pulled) and start the app:

   ```bash
   cd metro-ai-suite/metro-vision-ai-app-recipe/
   export SUPASS=$(cat ./smart-intersection/src/secrets/supass)
   docker compose build node-red
   docker compose up -d
   ```

4. Verify: `docker compose ps` should show `dlstreamer-pipeline-starter` exit `0` after it
   finishes staggering the 4 pipeline starts, and `curl -k https://localhost/api/pipelines/status`
   should show all 4 pipelines `RUNNING` rather than `ERROR`.
