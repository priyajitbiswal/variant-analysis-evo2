## Evo2 Variant Analysis (Variant effect prediction)

This repository contains a small web app for predicting the likely functional impact of single-nucleotide variants (SNVs) using **Evo2** and comparing the result to **ClinVar** classifications.

The system is split into:
- `evo2-backend/`: a **Modal** app that exposes a **FastAPI-style endpoint** for scoring variants on an H100-class GPU.
- `evo2-frontend/`: a **Next.js** (T3 stack) UI that lets users pick a genome assembly, browse/search genes, view reference sequence, and analyze variants.

## What the app does

1. User selects a genome assembly (for example `hg38`) and opens a gene.
2. The UI fetches the gene’s genomic bounds and reference sequence window using the **UCSC Genome API**.
3. The UI also fetches known variants in the selected region from **ClinVar** via **NCBI E-utilities**.
4. When the user clicks a nucleotide (or analyzes a known SNV), the frontend calls the backend endpoint with:
   - `variant_position` (1-based genomic coordinate)
   - `alternative` (single nucleotide `A/C/G/T`)
   - `genome` (UCSC genome assembly id, e.g. `hg38`)
   - `chromosome` (e.g. `chr17`)
5. The backend:
   - retrieves an ~`8192 bp` window around the position from UCSC,
   - scores the reference allele window and the alternative allele window with **Evo2**,
   - computes `delta_score = score(variant) - score(reference)`,
   - converts `delta_score` into a coarse class (`Likely pathogenic` vs `Likely benign`) and a heuristic confidence score.
6. The UI displays the prediction. For known ClinVar SNVs in the gene’s selected region, the UI can also run Evo2 for each variant and show a side-by-side comparison (agreement vs. disagreement) in a modal.

## Repository layout

- `evo2-backend/main.py`: Modal app + endpoint (`Evo2Model.analyze_single_variant`)
- `evo2-backend/requirements.txt`: Python deps for the Modal service
- `evo2-frontend/`: Next.js frontend (env var documented below)

## Backend (Modal) setup

### Requirements
- Python installed locally (used for `modal run/deploy`)
- A Modal account and authentication via the Modal CLI (`modal setup`)
- The backend runs Evo2 inference inside Modal’s GPU image.

### Install dependencies

From the repository root:

```bash
cd evo2-backend
pip install -r requirements.txt
```

### Configure Modal

```bash
modal setup
```

### Run and/or deploy

You can test quickly in development:

```bash
modal run main.py
```

For the frontend to work, deploy the endpoint:

```bash
modal deploy main.py
```

After deployment, you must set the deployed endpoint’s base URL in the frontend env var:
- `Evo2Model.analyze_single_variant` is exposed via `@modal.fastapi_endpoint(method="POST")`

### Endpoint contract

The frontend currently sends the request values as **query parameters on a POST** request (see `evo2-frontend/src/utils/genome-api.ts`).
Expected parameters:
- `variant_position` (number)
- `alternative` (string; single nucleotide)
- `genome` (string; e.g. `hg38`)
- `chromosome` (string; e.g. `chr17`)

Response JSON includes:
- `position`
- `reference`
- `alternative`
- `delta_score` (number)
- `prediction` (string; `Likely pathogenic` or `Likely benign`)
- `classification_confidence` (number between 0 and 1)

Note: the backend caches Hugging Face downloads in a Modal volume mounted at `/root/.cache/huggingface`.

## Frontend setup (Next.js)

### Install dependencies

```bash
cd evo2-frontend
npm i
```

### Configure environment variables

Create an `.env` file from `.env.example`:

```bash
copy .env.example .env
```

Set:
- `NEXT_PUBLIC_ANALYZE_SINGLE_VARIANT_BASE_URL` = the deployed backend endpoint base URL

### Run the app

```bash
npm run dev
```

Open the URL printed by the dev server (typically `http://localhost:3000`).

## Data sources

This app relies on public APIs for reference and annotations:
- UCSC Genome API: genome list/chromosomes and reference sequence windows
- NCBI E-utilities: gene metadata and ClinVar variant summaries

## Notes / limitations

- The scoring approach is window-based (fixed window size around the variant position).
- The decision threshold and confidence scaling used to map `delta_score` to classes are currently **hard-coded** in the backend.
- External API calls (UCSC/NCBI) can fail or be rate-limited; the UI will show an error if requests fail.
- This is intended for research/education and is not medical advice.

## References

- Evo2 paper: [Genome modeling and design across all domains of life with Evo 2](https://www.biorxiv.org/content/10.1101/2025.02.18.638918v1)
- Evo2 repository: [ArcInstitute/evo2](https://github.com/ArcInstitute/evo2)
