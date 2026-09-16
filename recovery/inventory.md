# Repository Inventory

## Entry Points

- Frontend app: `kitab-shop-fe/src/main.jsx` mounts the React/Vite app and `kitab-shop-fe/src/App.jsx` defines customer, account, and admin routes.
- Frontend scripts: `kitab-shop-fe/package.json` exposes `dev`, `build`, `lint`, and `preview`.
- Backend service: `kitab-shop-be/src/index.js` starts the Express API, loads `.env`, connects MongoDB, starts stock reservation cleanup, and registers `/api/v1/*` routes.
- Backend scripts: `kitab-shop-be/package.json` exposes `start`, `dev`, migration/seed scripts, and regression test scripts under `kitab-shop-be/tests`.

## Configuration

- Frontend config: `kitab-shop-fe/vite.config.js`, `kitab-shop-fe/tailwind.config.js`, `kitab-shop-fe/postcss.config.js`, `kitab-shop-fe/eslint.config.js`, `kitab-shop-fe/vercel.json`.
- Frontend env templates: `kitab-shop-fe/.env.example`, `kitab-shop-fe/.env.production`.
- Backend config: `kitab-shop-be/.env.example`, `kitab-shop-be/.env`, `kitab-shop-be/.env.production`, plus runtime config modules in `kitab-shop-be/src/config/`.
- Deployment and operational notes already exist under `docs/`, including `docs/000_deployment-runbook.md`, `docs/001_env-toggles.md`, and payment/order flow docs.

## Data And Seed Artifacts

- Seed/reference data lives in `kitab-shop-be/scripts/data/reference-books.js` and `kitab-shop-be/scripts/data/reference-genres.js`.
- Frontend static/demo data lives in `kitab-shop-fe/src/data/`.
- No local database dump was found in the repository inventory.

## Model Weights

- No ML model weight files such as `.pt`, `.onnx`, `.safetensors`, `.bin`, or checkpoints were found in the repository inventory.

## Output Artifacts

- Backend upload storage is configured through `kitab-shop-be/src/config/storage.config.js`.
- Test and audit artifacts exist as markdown under `docs/` and executable regression checks under `kitab-shop-be/test/regression/`.
- Frontend end-to-end helpers are under `kitab-shop-fe/e2e/`.

