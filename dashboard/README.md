# PointMatrix Frontend

React + Three.js dashboard for recorded and uploaded LiDAR scans.

```bash
npm ci
npm run dev
```

Configuration (`.env.production` for builds, `.env.example` as a template):

| Variable | Purpose |
| --- | --- |
| `VITE_FRAME_MANIFEST_URL` | Recorded sequence manifest (the Hugging Face dataset) |
| `VITE_HF_SPACE` | Hugging Face Space that processes uploads |
| `VITE_API_BASE_URL` | FastAPI backup in `../backend`, used only when `VITE_HF_SPACE` is empty |

Uploads go to the Space through `@gradio/client` (`src/hfSpace.js`); results are LGF1 frames
decoded by `src/lgf.js`. `npm test` runs the unit tests; `npm run build` writes `dist/`.
See the root README for deployment, views, and upload formats.
