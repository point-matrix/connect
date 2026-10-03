# PointMatrix

Web dashboard for adaptive 2.5D semantic mapping of LiDAR scans: six synchronized views of
raw points, semantic segmentation, the semantic 2.5D grid, adaptive resolution, elevation,
and drivability.

| Part | Where it runs |
| --- | --- |
| Website (`dashboard/`) | Vercel, static |
| Recorded sequence (KITTI seq. 08, frames 001500–002499, LGF1 format) | Hugging Face dataset [`ranbyDipz/sih-lidar-clip`](https://huggingface.co/datasets/ranbyDipz/sih-lidar-clip) |
| Uploads (SalsaNext + adaptive grid) | Hugging Face ZeroGPU Space set by `VITE_HF_SPACE` (currently [`RandomBengaliGuy/SIH`](https://huggingface.co/spaces/RandomBengaliGuy/SIH)) |
| Backup upload backend | FastAPI in `backend/` (local or Render), used only when `VITE_HF_SPACE` is empty |

The browser reads the recorded frames from the dataset and sends uploads to the Space directly
through [`@gradio/client`](https://www.gradio.app/guides/getting-started-with-the-js-client)
(`dashboard/src/hfSpace.js`). The live site needs no server of its own.

## Deploy on Vercel

1. Import the repository in Vercel with the repo root as the root directory. `vercel.json`
   sets the install/build commands and `dashboard/dist` as the output.
2. No environment variables are required: `dashboard/.env.production` sets
   `VITE_FRAME_MANIFEST_URL` (the dataset) and `VITE_HF_SPACE` (the Space). Variables set in
   the Vercel project override them.
3. Deploy. `VITE_*` values are fixed at build time, so redeploy after changing them.

## Local development

```bash
npm --prefix dashboard ci
npm --prefix dashboard run dev
```

`dashboard/.env.example` lists the variables.

## Dashboard

**Views.** Raw LiDAR points (coloured by height), Semantic Segmentation (points coloured by
class), Semantic 2.5D Map, Adaptive Resolution Map, Elevation Map, and Drivability Map. The
**Large panel** selector chooses which view fills the main panel (default: Semantic 2.5D Map).
Map views switch between isometric and top-down; every view has zoom, reset, and PNG capture.
Click a cell in a map view to inspect it.

**Playback.** Each window has its own frame, play/pause, speed (0.5×–4×), and zoom. The first
frame shows as soon as the manifest loads; later frames stream from the dataset in the
background and are prefetched ahead of each playing window. An upload is a single frame.

**Side panel.** System metrics for every view (frame, rendered points or cells, input points,
latency, playback state); layer controls (2.5D columns, cell boundaries); display controls
(ego marker, central 60 m detail view or full scan, block or exact-cell rendering of the
semantic map, point size); and the drivability profile.

**Semantic map rendering.** *Aggregated display blocks* (default) group nearby cells of the same
class into larger blocks and draw stylized vehicle shapes for separated vehicle clusters; these
are visual estimates, not object detections. *Exact cells* draws every grid cell with its real
footprint and height. Neither changes the grid, metrics, or exports.

**Ego marker.** The cyan "OUR CAR" marker sits at the sensor origin (x = 0, y = 0) with an arrow
along +X (forward). Frames are sensor-relative, so the scene moves around it during playback.
Its size is symbolic.

**Resolution.** Grid cells are 5, 10, 20, or 40 cm: finer near the vehicle and around obstacles,
coarser on open ground far away.

**Drivability** uses the rules in `dashbard_timesxript_drivability/drivability.ts`: semantic
class, terrain complexity, and confidence decide drivable (green), caution (amber), or blocked
(red); areas with no LiDAR returns are unknown, never drivable. Profiles: on-road (default) and
off-road. Default thresholds: caution at roughness 0.5, blocked at 0.8, minimum confidence 0.5;
the sliders change them live without reprocessing.

**Data shown.**
- Recorded frames are the dataset's "lite" files: every 4th point for display, all grid cells.
  Their timings are those measured when the clip was exported on a Kaggle Tesla T4.
- Uploads display up to 30,000 points, but every point goes through the pipeline. Their latency
  is the Space's own server time (GPU hand-over, SalsaNext on the Space GPU, grid engine in NumPy
  on CPU). It does not include upload/download time and is not the CUDA real-time benchmark.
  When the visitor's ZeroGPU quota is used up, the Space runs SalsaNext on CPU instead (slower).

**Exports.**
- *Capture* saves a PNG of a view.
- *Export JSON* (semantic and drivability views) saves the grid cells with per-cell
  drivability, the active profile/thresholds, drivable areas, and metrics. Classes in this
  JSON use the dashboard convention 0–18, with −1 = unlabeled.
- After an upload, *Download frame* saves the full result (`.lgf.gz`, every point and cell).

## Upload formats

| Mode | File | Columns | Labels |
| --- | --- | --- | --- |
| Raw | `.bin` (KITTI float32) or `.npy` | x, y, z, intensity | Predicted by SalsaNext |
| Labeled | `.npy` | x, y, z, intensity, class | 0 = unlabeled, 1–19 = classes; SalsaNext is skipped |
| Legacy labeled | `.npy` | x, y, z, class (+ two optional columns) | 0–18, 0 = car; FastAPI backup only |

Coordinates are in metres. Limits: 32 MB, 500,000 points, finite coordinates within 10 km.
Pickled/object `.npy` arrays are rejected.

## FastAPI backup

Used only when `VITE_HF_SPACE` is empty. Requires Python 3.12+ and Node.js 22.12+.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m backend.setup_model
npm --prefix dashboard ci
VITE_HF_SPACE= npm --prefix dashboard run build
python -m uvicorn backend.app:app --host 0.0.0.0 --port 8000
```

Open http://localhost:8000; the backend serves the website and `/api`. For frontend
development against it, run `VITE_HF_SPACE= npm --prefix dashboard run dev` (Vite proxies
`/api` to port 8000).

`backend.setup_model` downloads the [official SalsaNext pretrained model](https://github.com/TiagoCortinhal/SalsaNext#pretrained-model),
verifies its checksum, and exports a weights-only checkpoint (models are not committed). To use
another checkpoint, set `SALSANEXT_WEIGHTS` and `SALSANEXT_REPO` and put `arch_cfg.yaml` beside
it. Raw uploads use the notebook's range projection, SalsaNext with KNN post-processing (CUDA
if available, otherwise CPU), and the notebook's NumPy grid engine.

Jobs run one at a time (at most two pending); results stay in memory for the four most recent
jobs and are lost on restart. Run a single worker. With this backend, *Download NPZ* replaces
*Download frame*.

**Render:** `render.yaml` defines the backend as a Python web service (CPU PyTorch + built
frontend). Sync the Blueprint from the connected repository and pick a plan with enough memory
for PyTorch.

## Tests

```bash
npm --prefix dashboard test
npm --prefix dashboard run build
pip install httpx
python -m unittest backend.test_pipeline -v
```
