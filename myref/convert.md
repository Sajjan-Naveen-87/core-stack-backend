# Procedure: Port the local geospatial-calculation layer from Python → OCaml

> Goal: extract the **local** geospatial calculations (the Python code that produces the
> vectors/rasters published to GeoServer) into a **standalone, open-source OCaml
> component**, while leaving GEE (cloud) and the GeoServer publish step as thin boundaries.
> Read `branches-flow-connection.md` first — it explains the data flow, the
> cloud-vs-local split, and why `feature/local-compute-station` is the base branch.

---

## Guiding principles

1. **Port only the local math.** Anything that calls `ee.*` (Google Earth Engine) is a
   remote service, not portable code. Keep it in Python at the edges, or replace its
   *inputs* with locally-downloaded base layers (the `feature/local-compute-station`
   approach already does this).
2. **Keep GeoServer as a boundary.** `utilities/geoserver_utils.py` is a REST client, not
   a calculation. Reimplement it in OCaml later (or keep calling it over HTTP) — it is not
   on the critical path for the math.0
3. **Verify numerically, layer by layer.** Each ported module must produce
   bit-or-tolerance-equal output (same GeoTIFF values / same shapefile attributes) versus
   the Python reference before it's considered done. Geospatial bugs are silent.
4. **One module at a time, easiest first.** Start with `rasterize_vector.py` (≈50 lines,
   no GEE, single clear contract), then `drainage_density.generate_vector`, then up.

---

## Phase 0 — Set up the working branch

```bash
cd /home/snaveen/Desktop/core-stack-backend
git fetch origin
# Inspect the base branch
git checkout -b ocaml-port origin/feature/local-compute-station
# Confirm the config seam exists
ls computing/config.yaml computing/config_loader.py computing/local_compute_helper.py
```

Why this branch: it isolates calculation under `computing/`, adds a config layer that
removes hardcoded paths, and its `*_local.py` modules are *actually local* (rasterio/
numpy/geopandas) rather than GEE calls. See `branches-flow-connection.md` §6.

Deliverable: a branch where `python -c "import computing.config_loader"` resolves paths
relative to `PROJECT_ROOT`.

---

## Phase 1 — Inventory and freeze the contracts

For each candidate module (start with the §4 list in `branches-flow-connection.md`),
write down its **contract** so the OCaml version can be checked against it:

For every target function record:

- **Inputs**: file formats + CRS (e.g. shapefile EPSG:4326, attribute column `DD`).
- **Operation**: the exact transform (reproject → clip → length-sum → formula).
- **Outputs**: file format, CRS, dtype, resolution, nodata/fill value.
- **Numeric formula**: copy it verbatim. Example, drainage density:
  `DD = total_length_stream_order_km × influence_factor × 100 / area_in_ha/100`,
  with `influence_factors = [60/385, 55/385, …, 10/385]` for stream orders 1–11.

Priority order (lowest risk first):

1. `computing/clart/rasterize_vector.py` — vector+attribute → 30 m GeoTIFF.
2. `computing/clart/drainage_density.py::generate_vector` — the DD math.
3. `computing/utils.py` format/geometry helpers (geojson↔shp↔gpkg, `buffer(0)`).
4. `plans/build_layer.py` — CSV lat/lon → point shapefile.
5. The `*_local.py` raster modules (lulc, terrain, change detection) — larger, do last.

Deliverable: `myref/contracts/<module>.md` per module (or one table), plus a saved
reference output for each (the Python result on a fixed test tehsil, e.g. assam/baksa/baksa).

---

## Phase 2 — Choose the OCaml geospatial stack

The OCaml core must cover the primitives in `branches-flow-connection.md` §4. Candidate
libraries (verify availability/maturity when you start — confirm via opam, don't assume):

| Need                                           | OCaml option(s)                                         | Fallback                           |
| ---------------------------------------------- | ------------------------------------------------------- | ---------------------------------- |
| GeoTIFF / raster I/O                           | bindings to GDAL (`ocaml-gdal` if maintained)         | FFI to libgdal via `ctypes`      |
| Vector I/O (shp/gpkg/geojson)                  | GDAL/OGR via the same binding;`geojson` libs for JSON | shell out to `ogr2ogr` initially |
| CRS reprojection                               | PROJ via `ctypes` FFI to libproj                      | precompute transforms              |
| Geometry ops (clip, length, area, buffer, PIP) | a GEOS binding via `ctypes` to libgeos                | port simple ops natively           |
| Numeric arrays / raster math                   | `owl` (numpy-equivalent)                              | `bigarray`                       |
| YAML config (mirror config.yaml)               | `yaml` (ocaml)                                        | `ezjsonm`                        |

Recommended pragmatic path: **wrap the C libraries everyone already uses (GDAL, GEOS,
PROJ) via OCaml `ctypes`**, rather than reimplementing geometry algorithms. This keeps
numeric results identical to the Python/`shapely`/`rasterio` reference (which also wrap
GEOS/GDAL), so Phase 4 verification can hit tight tolerances.

Deliverable: a new opam project (e.g. `corestack-geocompute/`) with a `dune-project`,
a `geo` library exposing read/write/reproject/clip/length/area/rasterize, and a smoke
test that round-trips a GeoJSON.

---

## Phase 3 — Port module by module

For each module, in priority order:

1. **Implement** the OCaml equivalent behind a small CLI (e.g.
   `corestack-geocompute rasterize --in x.shp --col DD --out x.tif --res 0.000278`).
   Keep the same resolution/transform/fill conventions as the Python
   (`from_origin(minx, maxy, res, res)`, `all_touched=true`, fill `0`, dtype float32).
2. **Match the formula exactly.** For drainage density: same EPSG:7755 reprojection for
   length, same per-stream-order loop, same influence factors, same `/1000` km and
   `/100` ha conversions.
3. **Keep the file contract identical** (CRS tags, band count, nodata) so downstream
   `geoserver_utils.py` can publish the OCaml output unchanged.

Deliverable: one OCaml CLI subcommand (or library function) per ported module.

---

## Phase 4 — Verify against the Python reference

This is the gate; do not skip it.

1. Run the Python module and the OCaml module on the **same fixed input** (the test
   tehsil from Phase 1).
2. Compare outputs:
   - **Rasters**: compare with `gdalcompare.py` or read both with rasterio and assert
     `np.allclose(a, b, atol=…)`; check shape, transform, CRS, nodata.
   - **Vectors**: compare attribute tables (e.g. `DD`, `str_len_km`) within tolerance and
     geometry equality (GEOS `equals_exact` with a small tolerance).
3. Record max/mean absolute difference per layer. Tighten until differences are
   floating-point noise (CRS/clip ops on the same GEOS/GDAL should match closely).

Deliverable: a `verify/` script + a results table (module → max abs diff → pass/fail).

---

## Phase 5 — Wire the OCaml core back into the pipeline

You have two integration options; pick per how much you want to keep in Python short-term:

- **Option A (subprocess, lowest risk):** the Celery task in
  `computing/<theme>/<script>.py` shells out to the OCaml CLI for the math step, then
  hands the produced file to the existing `utilities/geoserver_utils.py` publish step.
  Minimal Django changes; the config seam from `feature/local-compute-station` already
  resolves the paths.
- **Option B (service):** expose the OCaml core as a small HTTP service; Django calls it.
  Better for the standalone open-source story, more work.

Keep GEE (`utilities/gee_utils.py`) and PostgreSQL metadata writes
(`computing/utils.py::save_layer_info_to_db`) in Python — they are integration glue, not
geospatial math, and don't need porting for the open-source component.

Deliverable: at least one end-to-end layer (drainage density recommended) generated
through the OCaml core and published to a local GeoServer, matching the Python output.

---

## Phase 6 — Package as a standalone open-source component

1. **Extract** the OCaml project (`corestack-geocompute/`) into its own repository, with
   no Django dependency — it reads inputs from disk (paths/config mirroring
   `computing/config.yaml`) and writes standard GeoTIFF/shapefile/GeoPackage.
2. **License**: match or stay compatible with the parent repo's `LICENSE`; add
   `LICENSE`, `README`, and a `CONTRIBUTING` to the new repo.
3. **Document the contracts** ported in Phase 1 as the public API spec.
4. **CI**: build via dune + run the Phase 4 verification fixtures as regression tests.
5. **Reference it back** from core-stack-backend (git submodule, opam pin, or released
   binary) so the Django app consumes it as Option A/B above.

Deliverable: a public OCaml repo that, given the same base layers, reproduces the local
geospatial layers CoRE Stack currently computes in Python.

---

## Scope guardrails (what NOT to port)

- ❌ Earth Engine math (`ee.Image`, `ee.Terrain.slope`, `ee.Image.pixelArea`, focal
  kernels) — remote service. Either keep in Python or replace its inputs with local base
  rasters (already done on `feature/local-compute-station`).
- ❌ `utilities/geoserver_utils.py` / `geoserver_styles.py` — REST publishing + SLD, not
  calculation. Port last, if at all.
- ❌ Django models, Celery wiring, JWT/API-key auth, STAC catalog, DPR/ODK — application
  glue, out of scope for the math component.

## Suggested first commit

Port `computing/clart/rasterize_vector.py` to an OCaml CLI, verify byte/tolerance-equal
GeoTIFF on assam/baksa/baksa, and document the contract. It is the smallest fully-local,
GEE-free, single-responsibility module — the ideal proof of concept before tackling
drainage density and the larger `*_local.py` raster modules.
