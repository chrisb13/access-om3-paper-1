---
render_macros: false
---

# ESMValTool recipe support: implementation review

!!! warning "Preview only: draft pull requests, not merged"
    This page and the recipe figures exist only on a `preview/` branch of a fork. They show the changes proposed in draft
    pull requests to [ACCESS-NRI/access-model-mkfigs](https://github.com/ACCESS-NRI/access-model-mkfigs/pulls?q=is%3Apr+esmvaltool) and
    [ACCESS-Community-Hub/access-om3-paper-1](https://github.com/ACCESS-Community-Hub/access-om3-paper-1/pulls?q=is%3Apr+esmvaltool). Together they let mkfigs run
    [ESMValTool](https://docs.esmvaltool.org/) recipes on ACCESS-OM3 output and show their figures next to the notebooks.
    Please comment on those pull requests.

!!! note "Written with Claude: please check"
    The code and this page were written mostly by Claude with feedback from @chrisb13.

## What it does

Five test recipes of increasing difficulty ran end to end on Gadi for `MC_25km_jra_iaf+wombatlite-test5.1-8f50b769` (2000–2002 or 2000–2004). They test the
pipeline, not the model, but most of them compare ACCESS-OM3 with observations and with other ACCESS models:

| Level | Recipe | What it exercises | Compared with |
|---|---|---|---|
| 1 | [`om3_mlotst_basic`](experiments/MC_25km_jra_iaf%2Bwombatlite-test5.1-8f50b769/om3_mlotst_basic.md) | one 2D variable, ESMValTool's example script | (model only) |
| 2 | [`om3_surface_fluxes`](experiments/MC_25km_jra_iaf%2Bwombatlite-test5.1-8f50b769/om3_surface_fluxes.md) | four 2D fields (two on u/v points), `areacello` for native-grid area means, ESMValTool's ocean scripts | ACCESS-ESM1-5; ACCESS-OM2 and -025 in the maps |
| 3 | [`om3_temp_salt_vs_obs`](experiments/MC_25km_jra_iaf%2Bwombatlite-test5.1-8f50b769/om3_temp_salt_vs_obs.md) | 3D `thetao`/`so` on z levels, observations | WOA18, EN4, ACCESS-ESM1-5; ACCESS-OM2 and -025 for salinity |
| 4 | [`om3_arctic_ocean`](experiments/MC_25km_jra_iaf%2Bwombatlite-test5.1-8f50b769/om3_arctic_ocean.md) | an upstream ESMValTool recipe with only its datasets swapped | PHC3, ACCESS-ESM1-5, ACCESS-OM2, ACCESS-OM2-025 |
| 5 | [`om3_russell18jgr_subset`](experiments/MC_25km_jra_iaf%2Bwombatlite-test5.1-8f50b769/om3_russell18jgr_subset.md) | NCL diagnostics (Russell et al. 2018 Figs 2, 3b, 3b-2) | ACCESS-ESM1-5, ACCESS-OM2, ACCESS-OM2-025 |

Each recipe gets a Figures page and a Recipe page (the resolved recipe, tool versions and the commands that reproduce the
plots). On the home page they are listed under *ESMValTool recipes*, in the sub-groups set by `RECIPE_CATEGORIES` in
`notebooks/mkfigs_issues.py`.

1. `qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh` runs `python -m mkfigs.run --kind recipes`.
2. For each recipe, `python -m mkfigs.moppy_prepare` CMORises the ACCESS-OM3 output it needs (MOPPy plus the workarounds
   below) into a per-experiment cache and writes an ESMValCore data config; ESMValCore's `data-hpc-nci` preset adds
   NCI's CMIP6 and observation collections.
3. `esmvaltool run` makes the plots (killed after `RECIPE_TIMEOUT` minutes, default 120, because ESMValTool can hang after
   a diagnostic script crashes); mkfigs copies them to `mkmd/<recipe>_NN.png` with `<recipe>.md`, a run marker and
   ESMValTool's resolved recipe.
4. `python -m mkfigs.pushit` adds the recipes to the home-page table and writes their Figures and Recipe pages.

Notebooks and recipes write into the same `mkfigs_output_<ENAME>/`, so one pushit publishes both. A plain `qsub mkfigs.sh`
still runs notebooks only.

## ACCESS-MOPPy workarounds

ACCESS-MOPPy 1.9 supports only ACCESS-ESM1.6 in its ESMValTool integration. `mkfigs.moppy_prepare` patches six things in
memory and then calls MOPPy's own `moppy-esmval-prepare` code. Each one is an upstream MOPPy fix to ask for; its wrapper
code goes once MOPPy has it.

??? info "1. Recipe datasets: MOPPy only CMORises ACCESS-ESM1-5/1-6"
    MOPPy's recipe reader drops other dataset names and its command line can't change that, though its Python API can
    (`prepare_recipe(allowed_datasets=...)`). The wrapper passes the model id (`MOPPY_MODEL_ID=ACCESS-OM3` in `mkfigs.sh`).

??? info "2. Model list: the CMIP6 source_id list has no ACCESS-OM3"
    The wrapper adds a provisional entry copied from ACCESS-OM2, with the ocean and sea-ice resolution set from
    `MOPPY_NOMINAL_RES` (25 km) and ACCESS-OM2's component descriptions (MOM5, CICE5) replaced by a "provisional" note,
    since they end up in each file's `source` attribute. The institution list has no ACCESS-NRI either, so the recipes say
    `CSIRO-COSIMA`.

??? info "3. Ocean grid: MOPPy's bundled 25 km supergrid is ACCESS-OM2's"
    That grid has 1080 rows; ACCESS-OM3 25 km has 1152. The wrapper uses the run's own `ocean_hgrid.nc`
    (`MOPPY_SUPERGRID`, default the `global.25km/2026.06.11` grid in vk83).

??? info "4. Creator details: MOPPy asks on stdin when ~/.moppy/user.yml is missing"
    In a PBS job that ends in `EOFError`. The wrapper writes a generic profile to `mkfigs_output_<ENAME>/moppy_user.yml`
    (or uses `MKFIGS_MOPPY_USER_CONFIG`). Its values end up in the CMORised files' global attributes.

??? info "5. Version pinning broke comparisons with published ACCESS-ESM1-5 data"
    MOPPy writes a `<recipe>.moppy-pinned.yml` copy that sets `version:` on every CMIP6 entry named ACCESS-ESM1-5/1-6 (a
    fixed list), not on the datasets it CMORised. ESMValTool then looked for the published ACCESS-ESM1-5 files under the
    date of the CMORisation and found nothing, and every comparison recipe failed. Pinning ACCESS-OM3 instead isn't safe
    either (MOPPy pins one version per recipe, and a cache reused across days has several), so the wrapper turns pinning
    off; ESMValCore already uses the newest version of each variable.

??? info "6. 3D fields on z levels: the depth axis was not CMORised"
    MOPPy's ACCESS-OM3 mapping knows MOM6's native vertical axis `zl`, but the z-remapped `+z` diagnostics use `z_l`/`z_i`.
    MOPPy left `z_l` as it was (units "meters", no bounds) and moved it last (`so(time, j, i, z_l)`), and ESMValCore
    rejected the files ("olevel: does not exist"). The wrapper reads `z_l`/`z_i` as `zl`/`zi`, giving `so(time, lev, j, i)`
    with a CMOR depth axis. Still missing upstream: `lev_bnds` (ESMValCore accepts files without them), because the
    ACCESS-OM3 mapping names the axis `depth_coord` where MOPPy's bounds code expects `lev`, as in the ESM1.6 mapping.

??? info "Raw-file patterns (no patch needed)"
    MOPPy's default globs follow the ESM1.6 archive; ACCESS-OM3 writes `outputNNN/access-om3.mom6.2d.<var>.1mon.mean.<YYYY>.nc`.
    `MOPPY_PATTERNS` in `mkfigs.sh` passes MOPPy's own `--pattern COMPOUND_NAME:GLOB`, one per variable. MOPPy only reads a
    recipe's `variables:`, not `supplementary_variables:`, so a recipe that needs ACCESS-OM3's `areacello` lists it as a
    variable (see `om3_surface_fluxes`).

## Test run on Gadi

| | |
|---|---|
| PBS job | 183668259 (normal queue, 16 CPUs, 190 GB; earlier attempts 183654445 and 183666762 found the bugs listed below) |
| Submitted as | bash mkfigs.sh with MKFIGS_KIND=recipes, MKFIGS_CONDA_MODULE=conda/analysis3-26.11, MKFIGS_ESMVALTOOL_ARGS=--max-parallel-tasks 4 |
| Walltime | 53 min for all five, peak memory 64 GB: om3_mlotst_basic 45 s, om3_surface_fluxes 1.7 min, om3_temp_salt_vs_obs 12.7 min (incl. CMORising 3 years of 3D thetao and so), om3_arctic_ocean 35 min, om3_russell18jgr_subset 2.5 min |
| Result | 5 of 5 OK: om3_mlotst_basic 2 plots, om3_surface_fluxes 20, om3_temp_salt_vs_obs 20, om3_arctic_ocean 25, om3_russell18jgr_subset 3 |
| Versions | ESMValTool 2.15.0, ESMValCore 2.15.0, ACCESS-MOPPy 1.9.0 |
| Comparison data | ACCESS-ESM1-5 historical r1i1p1f1 (2000-2002) and ACCESS-OM2 / ACCESS-OM2-025 omip2 (model years 0348-0350 = 2000-2002), CMIP6 on fs38/oi10; WOA18, EN4, PHC3 from the ESMValTool observation replica (ct11) |

Unit tests: 149 passed, 1 file skipped (conda/analysis3-26.08); the skipped file's 9 tests pass against MOPPy 1.9.0 in
conda/analysis3-26.11.

??? bug "Problems found on the way (all fixed or worked around)"
    - **ESMValTool's ocean diagnostic scripts** (`ocean/diagnostic_*.py`) read `documentation: projects:` and crash without
      it; the recipes now have `projects: []`.
    - **ESMValTool hangs** instead of exiting when a diagnostic script crashes or is killed (memory), holding the job until
      walltime. mkfigs now kills a recipe after `RECIPE_TIMEOUT` minutes (the whole process group) and moves on.
    - **Preprocessor order.** ESMValCore's default order runs `extract_levels` and `regrid` before `climate_statistics`,
      i.e. on every monthly 3D field (over an hour for one variable). The recipes use `custom_order: true`, time mean first.
      `ocean/diagnostic_profiles.py` needs a time axis, so its preprocessor uses `annual_statistics`.
    - **NCL out of memory.** Russell et al. Figs 3b/3b-2 have no preprocessor upstream, so NCL loaded every monthly 3D field
      (over 32 GB for 1-degree ACCESS-OM2). The subset recipe keeps the top 500 m and annual means.
    - **Published ACCESS-OM2 and ACCESS-OM2-025 omip2 `thetao`** (fs38, v20210616/v20210617) holds Kelvin values labelled
      `degC`. ESMValCore has no fix, so level 3 compares those two models for salinity only. The Arctic and Russell scripts
      convert Kelvin themselves.
    - **Time axes.** OMIP models count model years (0348 = 2000 in the last JRA55-do cycle). ESMValCore can't shift a time
      axis, so ACCESS-OM2(-025) appear only in time-mean plots, not in multi-model time series.
    - A stale `<recipe>.moppy-pinned.yml` from an earlier run would have been run instead of the current recipe; it is now
      removed first.

## Decisions to make

!!! question "Open questions"
    1. **Institution.** Keep `institute: CSIRO-COSIMA` in recipes, or ask MOPPy for an ACCESS-NRI entry?
    2. **Creator profile.** The automatic profile names access-model-mkfigs with a placeholder email. Use a real ACCESS-NRI
       contact, or require each user's own `~/.moppy/user.yml`?
    3. **Upstream.** One MOPPy issue per workaround (now six), or one "ACCESS-OM3 support" issue? And report the
       ACCESS-OM2 `thetao` units to the CMIP6 errata service?
    4. **Which recipes for real?** These five are pipeline tests. `russell18jgr` in full needs ESMValTool PR 4612 (not in
       2.15.0).
    5. **ACCESS-ESM1-6.** There is no CMORised ACCESS-ESM1-6 output on Gadi yet (only raw output in p73). Comparing with it
       would mean CMORising it with MOPPy too (its native case). Worth it now?

!!! success "Decided (2026-11-11)"
    - Recipe sub-groups come from a `RECIPE_CATEGORIES` list in `mkfigs_issues.py`, not from subfolders.
    - The MOPPy cache is per experiment: `/g/data/tm70/$USER/moppy-esmval-cache/<ENAME>` (MOPPy writes every ACCESS-OM3
      run to the same CMIP6 path, so a shared cache mixed runs).

??? note "Preview quirks (not from this change)"
    - The home page's *ESM datastore* line shows the path of the last active experiment in `mkfigs.sh`, while this preview
      was built for test5.1. The notebooks show *Not run* because this preview has no test5.1 notebook outputs.
    - `pages/experiments/MC_25km_jra_iaf-1.0-beta-gm5-9b5dbfa9/timeseries.md` on main contains 784 NUL bytes (commit ccb7b95).

## Changed files

| Repo | File | State | What |
|---|---|---|---|
| `model-mkfigs` | `src/mkfigs/moppy_prepare.py` | new | New. moppy-esmval-prepare plus six ACCESS-OM3 workarounds, each a candidate upstream MOPPy fix. |
| `model-mkfigs` | `src/mkfigs/recipes.py` | new | New (from the handover patch, fixed on Gadi). Runs one recipe: MOPPy, esmvaltool run (with an optional time limit), collect plots, run marker, Recipe page with reproduce commands. |
| `model-mkfigs` | `src/mkfigs/run.py` | modified | --kind notebooks|recipes|all; recipe loop; separate recipe error log. |
| `model-mkfigs` | `src/mkfigs/pushit.py` | modified | Recipes in their own landing-table section and nav group, optionally split by RECIPE_CATEGORIES; a Recipe page per recipe. |
| `model-mkfigs` | `src/mkfigs/configdoc.py` | modified | Figure markdown can say it came from a recipe rather than a notebook. |
| `model-mkfigs` | `tests/test_moppy_prepare.py` | new | New. Runs against the real access_moppy (skipped where MOPPy < 1.8). |
| `model-mkfigs` | `tests/test_recipes.py` | new | New. Runner, layout, pushit and run.main tests with a fake ESMValTool. |
| `om3-paper-1` | `notebooks/mkfigs.sh` | modified | Recipe list, MOPPY_* settings, per-experiment MOPPy cache, ESMValCore data preset, recipe time limit, storage flags. |
| `om3-paper-1` | `notebooks/mkfigs_issues.py` | modified | RECIPE_CATEGORIES: sub-groups for the recipes section. |
| `om3-paper-1` | `notebooks/recipes/om3_mlotst_basic.yml` | new | New. Level 1: mixed layer depth, model only. |
| `om3-paper-1` | `notebooks/recipes/om3_surface_fluxes.yml` | new | New. Level 2: four 2D fields incl. staggered tau, native-grid area means with areacello; vs ACCESS-ESM1-5, ACCESS-OM2(-025). |
| `om3-paper-1` | `notebooks/recipes/om3_temp_salt_vs_obs.yml` | new | New. Level 3: 3D thetao/so vs WOA18 and EN4, and vs ACCESS-ESM1-5, ACCESS-OM2(-025). |
| `om3-paper-1` | `notebooks/recipes/om3_arctic_ocean.yml` | new | New. Level 4: ESMValTool's recipe_arctic_ocean, datasets swapped (vs PHC3 and the three ACCESS models). |
| `om3-paper-1` | `notebooks/recipes/om3_russell18jgr_subset.yml` | new | New. Level 5: three NCL figures of recipe_russell18jgr (Southern Ocean wind stress and fronts). |
| `om3-paper-1` | `notebooks/recipes/README.md` | new | New. How to add and run a recipe; the workaround table. |

New files are shown in full, modified files as diffs. Click a file to open it.

??? abstract "access-model-mkfigs/src/mkfigs/moppy_prepare.py (new file, +235 −0)"

    New. moppy-esmval-prepare plus six ACCESS-OM3 workarounds, each a candidate upstream MOPPy fix.

    ```python
    # Copyright 2024 ACCESS-NRI and contributors. See the top-level COPYRIGHT file for details.
    # SPDX-License-Identifier: Apache-2.0
    
    """``moppy-esmval-prepare`` for models ACCESS-MOPPy's ESMValTool integration doesn't support yet (e.g. ACCESS-OM3).
    
    Usage (same arguments as ``moppy-esmval-prepare``, plus the mkfigs options below)::
    
        python -m mkfigs.moppy_prepare RECIPE --input-root ROOT --cache-dir CACHE --output-config CFG \\
            --model-id ACCESS-OM3 --pattern 'Omon.mlotst:output[0-9]*/access-om3.mom6.2d.mlotst.1mon.mean.*.nc' \\
            --supergrid /g/data/vk83/.../ocean_hgrid.nc --user-config moppy_user.yml
    
    ACCESS-MOPPy 1.9 (conda/analysis3-26.11) assumes ACCESS-ESM1.6 in six places. This module works around each one in
    memory, then hands over to MOPPy's own ``moppy-esmval-prepare`` code, so CMORisation, the ESMValCore data config and
    the ``<recipe>.moppy-pinned.yml`` copy are exactly what MOPPy produces. Every workaround is a candidate upstream fix;
    delete it here once MOPPy has it:
    
    1. Recipe datasets: ``RecipeReader`` keeps only ACCESS-ESM1-5/ACCESS-ESM1-6 and the CLI can't change that
       (``prepare_recipe(allowed_datasets=...)`` can) -> ``--dataset`` (default: ``--model-id``).
    2. Source CV: the CMIP6 ``source_id`` CV has no ACCESS-OM3 -> registered from a template entry
       (``--cv-template``, default ACCESS-OM2) with ``--nominal-resolution``. Skipped if the CV already has the model.
    3. Ocean supergrid: MOPPy picks its bundled grid by nominal resolution; its "25 km" grid is ACCESS-OM2's
       (1080 rows) but ACCESS-OM3 25 km has 1152 -> ``--supergrid`` (the run's own MOM6 ``ocean_hgrid.nc``).
    4. Creator metadata: MOPPy reads ``~/.moppy/user.yml`` and ``input()``-prompts when it is missing, which dies with
       EOFError in a batch job -> ``--user-config`` (created with generic mkfigs values if the file doesn't exist).
    5. Version pinning: the ``.moppy-pinned.yml`` copy pins ``version:`` on every CMIP6 entry named ACCESS-ESM1-5/1-6
       (a fixed list), not on the datasets MOPPy CMORised. So a recipe comparing with published ACCESS-ESM1-5 data asks
       for that data under today's CMORisation version and finds nothing. Pinning the CMORised datasets instead isn't
       safe either: MOPPy pins one version (the newest) per recipe, and a cache reused across days holds variables
       under different versions -> no pinning; ESMValCore already uses the newest version of each variable.
    6. 3D ocean output on z levels: the ACCESS-OM3 mapping knows MOM6's native vertical axis ``zl`` but not ``z_l``/``z_i``
       of the z-remapped ``+z`` diagnostics, so the depth axis was left un-CMORised (``z_l``, units "meters", no
       bounds) and ended up last (``so(time, j, i, z_l)``); ESMValCore rejects it ("olevel: does not exist") ->
       each input file's ``z_l``/``z_i`` are read as ``zl``/``zi``. Still upstream: the files get no ``lev_bnds``
       (ESMValCore accepts that), because the ACCESS-OM3 mapping names the axis ``depth_coord`` (the table key)
       where MOPPy's bounds lookup expects the out_name ``lev`` (as in the ESM1.6 mapping), and ``depth_edges`` is
       empty for ACCESS-OM3. Same for native ``zl`` output.
    
    Must run in a Python that has ``access_moppy`` >= 1.8 (the recipes conda environment; tested with 1.9.0 in
    conda/analysis3-26.11). Older MOPPy reads the creator profile when the package is imported, before workaround 4
    can redirect it, so it is refused rather than left to hang or die on an ``input()`` prompt.
    """
    
    from __future__ import annotations
    
    import argparse
    import copy
    import functools
    import logging
    import sys
    from pathlib import Path
    
    log = logging.getLogger("mkfigs.moppy_prepare")
    
    MIN_MOPPY_VERSION = (1, 8)
    
    DEFAULT_CREATOR = {
        "creator_name": "access-model-mkfigs",
        "organisation": "ACCESS-NRI",
        "creator_email": "noreply@example.invalid",  # MOPPy requires a non-empty value
        "creator_url": "https://github.com/ACCESS-NRI/access-model-mkfigs",
    }
    
    
    def build_parser() -> argparse.ArgumentParser:
        """Only the mkfigs options; everything else is passed through to ``moppy-esmval-prepare``."""
        p = argparse.ArgumentParser(
            prog="python -m mkfigs.moppy_prepare",
            description="moppy-esmval-prepare with mkfigs workarounds for non-ESM1.6 models. "
                        "Unrecognised arguments are passed to moppy-esmval-prepare.",
        )
        p.add_argument("--dataset", action="append", default=[], metavar="NAME",
                       help="Recipe dataset name to CMORise (repeatable; default: the --model-id value).")
        p.add_argument("--cv-template", default="ACCESS-OM2", metavar="SOURCE_ID",
                       help="CMIP6 source_id CV entry to copy when --model-id isn't in the CV (default ACCESS-OM2).")
        p.add_argument("--nominal-resolution", default="", metavar="RES",
                       help="Ocean/sea-ice nominal resolution for the registered CV entry, e.g. '25 km'.")
        p.add_argument("--supergrid", default="", metavar="FILE",
                       help="MOM supergrid (ocean_hgrid.nc) to use instead of MOPPy's bundled one.")
        p.add_argument("--user-config", default="", metavar="FILE",
                       help="MOPPy creator profile to use instead of ~/.moppy/user.yml (written if missing).")
        return p
    
    
    def moppy_version_ok() -> bool:
        """True if the installed ACCESS-MOPPy only reads its creator profile when CMORising (>= 1.8), not on import."""
        from importlib import metadata
    
        try:
            version = metadata.version("access-moppy")
        except metadata.PackageNotFoundError:
            return False
        parts = []
        for piece in version.split(".")[:2]:
            digits = "".join(ch for ch in piece if ch.isdigit()) or "0"
            parts.append(int(digits))
        return tuple(parts) >= MIN_MOPPY_VERSION
    
    
    def _model_id(passthrough: list[str]) -> str:
        for i, arg in enumerate(passthrough):
            if arg == "--model-id" and i + 1 < len(passthrough):
                return passthrough[i + 1]
            if arg.startswith("--model-id="):
                return arg.split("=", 1)[1]
        return ""
    
    
    def allow_datasets(names: frozenset[str]) -> None:
        """Workaround 1: make ``CMORiseOrchestrator.prepare_recipe`` accept *names* when the caller passes nothing."""
        from access_moppy.esmval.orchestrator import CMORiseOrchestrator
    
        original = CMORiseOrchestrator.prepare_recipe
    
        @functools.wraps(original)
        def prepare_recipe(self, recipe_path, allowed_datasets=None):
            return original(self, recipe_path, allowed_datasets=allowed_datasets or names)
    
        CMORiseOrchestrator.prepare_recipe = prepare_recipe
    
        # Workaround 5: no version pinning at all (MOPPy reads this module global at call time).
        from access_moppy.esmval import cli_commands
        cli_commands._ACCESS_DATASETS = frozenset()
    
    
    def register_source_id(model_id: str, template: str, nominal_resolution: str = "") -> bool:
        """Workaround 2: add *model_id* to MOPPy's CMIP6 source_id CV, copied from *template*. False if already there."""
        import json
    
        import access_moppy
        from access_moppy import vocabulary_processors as vp
    
        cv_file = Path(access_moppy.__file__).parent / "vocabularies" / "CMIP6_CVs" / "CMIP6_source_id.json"
        cv = json.loads(cv_file.read_text())["source_id"]
        if model_id in cv or model_id in vp._CMIP6_TEMP_SOURCE_OVERRIDES:
            return False
        if template not in cv:
            raise SystemExit(f"--cv-template {template!r} is not in MOPPy's CMIP6 source_id CV")
        entry = copy.deepcopy(cv[template])
        entry.update(source_id=model_id, label=model_id,
                     label_extended=f"{model_id} (provisional CV entry added by access-model-mkfigs from {template})")
        # The template's component descriptions name the template model (e.g. "ACCESS-OM2 (MOM5, ... 1deg ...)") and
        # MOPPy copies them into each file's `source` attribute: don't let them describe the wrong model.
        for component, info in entry.get("model_component", {}).items():
            if info.get("description", "none") != "none":
                info["description"] = (f"{model_id} {component} (provisional CV entry from access-model-mkfigs: "
                                       "see the experiment's configuration)")
            if nominal_resolution and component in ("ocean", "ocnBgchem", "seaIce"):
                info["native_nominal_resolution"] = nominal_resolution
        vp._CMIP6_TEMP_SOURCE_OVERRIDES[model_id] = entry
        return True
    
    
    #: MOM6 z-remapped (``+z``) axis names -> the native names ACCESS-MOPPy's ACCESS-OM3 mapping uses.
    Z_LEVEL_ALIASES = {"z_l": "zl", "z_i": "zi"}
    
    
    def _z_level_preprocess(ds, **kwargs):
        """MOPPy's per-file ``open_mfdataset`` preprocess, with ``z_l``/``z_i`` renamed first (workaround 6).
    
        Module-level (not a closure) so it stays picklable for Dask workers; it finds MOPPy's original through
        ``access_moppy.base`` either way (patched or not).
        """
        from access_moppy import base
    
        original = getattr(base, "_mkfigs_original_preprocess", base._preprocess_open_mfdataset)
        rename = {old: new for old, new in Z_LEVEL_ALIASES.items()
                  if (old in ds.variables or old in ds.dims) and new not in ds.variables and new not in ds.dims}
        if rename:
            ds = ds.rename(rename)
            edges_attr = ds["zl"].attrs.get("edges") if "zl" in ds.variables else None
            if edges_attr in Z_LEVEL_ALIASES:
                ds["zl"].attrs["edges"] = Z_LEVEL_ALIASES[edges_attr]
        return original(ds, **kwargs)
    
    
    def support_z_level_output() -> None:
        """Workaround 6: CMORise ACCESS-OM3 3D output written on z levels (``z_l``) like native ``zl`` output."""
        from access_moppy import base
    
        if not hasattr(base, "_mkfigs_original_preprocess"):
            base._mkfigs_original_preprocess = base._preprocess_open_mfdataset
            base._preprocess_open_mfdataset = _z_level_preprocess
    
    
    def use_supergrid(path: str) -> None:
        """Workaround 3: always use *path* as the ocean supergrid."""
        from access_moppy import ocean_supergrid
    
        if not Path(path).is_file():
            raise SystemExit(f"--supergrid file not found: {path}")
        ocean_supergrid.Supergrid.get_supergrid_path = lambda self, nominal_resolution: str(path)
    
    
    def use_user_config(path: str) -> None:
        """Workaround 4: read MOPPy's creator profile from *path*, writing generic mkfigs values first if it is missing."""
        import yaml
        from access_moppy import _config
    
        p = Path(path).expanduser()
        if not p.is_file():
            p.parent.mkdir(parents=True, exist_ok=True)
            p.write_text(yaml.safe_dump(DEFAULT_CREATOR, sort_keys=False))
            log.info("Wrote MOPPy creator profile %s", p)
        _config.CONFIG_PATH = str(p)
    
    
    def main(argv: list[str] | None = None) -> int:
        args, passthrough = build_parser().parse_known_args(argv)
        model_id = _model_id(passthrough)
        logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
        if not moppy_version_ok():
            log.error("ACCESS-MOPPy >= %s is required (e.g. module load conda/analysis3-26.11)",
                      ".".join(map(str, MIN_MOPPY_VERSION)))
            return 1
    
        datasets = frozenset(args.dataset or ([model_id] if model_id else []))
        if datasets:
            allow_datasets(datasets)
            log.info("CMORising recipe datasets: %s", ", ".join(sorted(datasets)))
        if model_id and register_source_id(model_id, args.cv_template, args.nominal_resolution):
            log.info("Registered provisional CMIP6 source_id %s (from %s)", model_id, args.cv_template)
        support_z_level_output()
        if args.supergrid:
            use_supergrid(args.supergrid)
            log.info("Using supergrid %s", args.supergrid)
        if args.user_config:
            use_user_config(args.user_config)
    
        from access_moppy.esmval.cli_commands import main_prepare
    
        return main_prepare(passthrough)
    
    
    if __name__ == "__main__":
        sys.exit(main())
    ```

??? abstract "access-model-mkfigs/src/mkfigs/recipes.py (new file, +514 −0)"

    New (from the handover patch, fixed on Gadi). Runs one recipe: MOPPy, esmvaltool run (with an optional time limit), collect plots, run marker, Recipe page with reproduce commands.

    ````python
    # Copyright 2024 ACCESS-NRI and contributors. See the top-level COPYRIGHT file for details.
    # SPDX-License-Identifier: Apache-2.0
    
    """ESMValTool recipe support for mkfigs (DRAFT: not yet run against real ESMValTool output).
    
    A *recipe* is an ESMValTool recipe YAML that mkfigs runs instead of a notebook.
    For each recipe stem ``<stem>`` the runner:
    
    1. copies ``<recipes dir>/<stem>.yml`` into ``mkfigs_output_<ename>/recipes/``;
    2. optionally CMORises ACCESS output on the fly with ACCESS-MOPPy's ``moppy-esmval-prepare``, run through
       :mod:`mkfigs.moppy_prepare` so non-ESM1.6 models such as ACCESS-OM3 work (only when an input root is given);
    3. runs ``esmvaltool run`` into ``mkfigs_output_<ename>/esmvaltool/<stem>/``;
    4. copies every plot into ``mkmd/<stem>_NN.png`` and writes ``mkmd/<stem>.md``
       (same shape MkmdWriter produces for notebooks, so pushit's Figshare upload,
       link rewriting and landing-page table work unchanged);
    5. copies ESMValTool's resolved ``run/<recipe>_filled.yml`` next to them and writes a small
       ``<stem>_recipe_run.json`` marker. The marker plays the role that
       ``<nb>_rendered.ipynb`` plays for notebooks: pushit uses it to tell OK / FAILED /
       NOT RUN apart, and the recipe page shows its tool versions.
    
    pushit later turns the marker + filled recipe into a ``recipes/<stem>.md`` page that embeds the recipe.
    
    ESMValTool's output layout, checked against a real ESMValTool 2.15 / ESMValCore 2.15 run on Gadi (2026-11-11):
    ``<recipe file stem>_<YYYYMMDD_HHMMSS>/`` (UTC timestamp) containing ``plots/<diag>/<script>/png/*.png`` and
    ``run/<recipe file stem>_filled.yml``. Upstream recipes are named ``recipe_<x>.yml``, hence ``recipe_<x>_<ts>/``.
    """
    
    from __future__ import annotations
    
    import json
    import importlib.util
    import os
    import logging
    import re
    import shlex
    import shutil
    import signal
    import subprocess
    import sys
    from datetime import datetime, timezone
    from importlib import metadata
    from pathlib import Path
    
    from .configdoc import _mkmd_notebook
    
    log = logging.getLogger(__name__)
    
    RUN_MARKER_SUFFIX = "_recipe_run.json"
    FILLED_SUFFIX = "_recipe_filled.yml"
    DEFAULT_MOPPY_CACHE = "~/.cache/moppy-esmval"
    TOOL_DISTRIBUTIONS = ("ESMValTool", "ESMValCore", "access-moppy")
    # ESMValCore 2.14+ reads every YAML in ESMVALTOOL_CONFIG_DIR (default ~/.config/esmvaltool). mkfigs keeps a per-output
    # config directory instead, so moppy-esmval-prepare's data-source file never lands in $HOME and runs stay reproducible.
    CONFIG_SUBDIR = "esmvaltool_config"
    MOPPY_CONFIG_NAME = "moppy-esmval-data.yml"
    # MOPPy creator profile, kept in the output folder (not $HOME, and not the ESMValTool config dir, which must only
    # hold ESMValCore config files).
    MOPPY_USER_CONFIG_NAME = "moppy_user.yml"
    # ESMValCore ships ready-made config files (data sources per HPC site, e.g. ``data-hpc-nci`` for Gadi's CMIP and
    # observation replicas). Recipes that compare against observations or other models need one of them.
    PRESETS_ENV = "MKFIGS_ESMVALTOOL_PRESETS"
    
    
    def marker_path(ofol: Path | str, stem: str) -> Path:
        return Path(ofol) / f"{stem}{RUN_MARKER_SUFFIX}"
    
    
    def filled_recipe_path(ofol: Path | str, stem: str) -> Path:
        return Path(ofol) / f"{stem}{FILLED_SUFFIX}"
    
    
    def read_marker(ofol: Path | str, stem: str) -> dict | None:
        """Return the run marker for *stem*, or None if the recipe was never run (or the marker is unreadable)."""
        path = marker_path(ofol, stem)
        try:
            return json.loads(path.read_text())
        except (OSError, json.JSONDecodeError):
            return None
    
    
    def recipe_stems_in(ofol: Path | str) -> list[str]:
        """Stems of every recipe that has a run marker in *ofol*."""
        ofol = Path(ofol)
        if not ofol.is_dir():
            return []
        return sorted(p.name[: -len(RUN_MARKER_SUFFIX)] for p in ofol.glob(f"*{RUN_MARKER_SUFFIX}"))
    
    
    _RUN_DIR_RE = re.compile(r"_\d{8}_\d{6}$")
    
    
    def find_run_dir(output_dir: Path) -> Path | None:
        """Newest ESMValTool run directory (``<recipe stem>_<YYYYMMDD_HHMMSS>``) under *output_dir*.
    
        *output_dir* is per recipe, so all runs share the ``<recipe stem>_`` prefix and the timestamp suffix
        makes the lexical order chronological.
        """
        runs = sorted(
            p for p in Path(output_dir).iterdir()
            if p.is_dir() and _RUN_DIR_RE.search(p.name) and (p / "plots").is_dir()
        ) if Path(output_dir).is_dir() else []
        return runs[-1] if runs else None
    
    
    def find_filled_recipe(run_dir: Path, recipe_stem: str) -> Path | None:
        """ESMValTool's resolved recipe, ``run/<recipe stem>_filled.yml`` (any ``*_filled.yml`` as a fallback)."""
        exact = Path(run_dir) / "run" / f"{recipe_stem}_filled.yml"
        if exact.is_file():
            return exact
        others = sorted((Path(run_dir) / "run").glob("*_filled.yml"))
        return others[0] if others else None
    
    
    # mkfigs.moppy_prepare options settable from mkfigs.sh: env var -> option (MKFIGS_MOPPY_DATASETS is ':'-separated).
    MOPPY_ENV_OPTIONS = {
        "MKFIGS_MOPPY_SUPERGRID": "--supergrid",
        "MKFIGS_MOPPY_NOMINAL_RES": "--nominal-resolution",
        "MKFIGS_MOPPY_WORKERS": "--workers",
        "MKFIGS_MOPPY_USER_CONFIG": "--user-config",
    }
    
    
    def moppy_args_from_env(environ=os.environ) -> list[str]:
        """Extra :mod:`mkfigs.moppy_prepare` options from the ``MKFIGS_MOPPY_*`` environment variables that are set."""
        args: list[str] = []
        for var, opt in MOPPY_ENV_OPTIONS.items():
            if environ.get(var, "").strip():
                args += [opt, environ[var].strip()]
        for name in filter(None, environ.get("MKFIGS_MOPPY_DATASETS", "").split(":")):
            args += ["--dataset", name.strip()]
        return args
    
    def _reset_previous_outputs(mkmd: Path, stem: str) -> None:
        """Remove this recipe's markdown and PNGs from an earlier run (exact match, so "MLD" never removes "MLD_max_01.png")."""
        own_png = re.compile(rf"^{re.escape(stem)}_\d+\.png$")
        for path in [mkmd / f"{stem}.md", *(p for p in mkmd.glob("*.png") if own_png.match(p.name))]:
            if path.exists():
                path.unlink()
                log.warning("Removed output from a previous run: %s", path)
    
    
    def collect_outputs(
        run_dir: Path,
        mkmd: Path,
        stem: str,
        experiment: str,
        repo_root: Path,
        nb_relpath: str | None = None,
    ) -> int:
        """Copy ESMValTool plots into *mkmd* as ``<stem>_NN.png`` and write ``<stem>.md``. Returns the plot count."""
        plots_root = Path(run_dir) / "plots"
        pngs = sorted(plots_root.rglob("*.png")) if plots_root.is_dir() else []
        mkmd.mkdir(parents=True, exist_ok=True)
        _reset_previous_outputs(mkmd, stem)
        for i, png in enumerate(pngs, start=1):
            fname = f"{stem}_{i:02d}.png"
            shutil.copy2(png, mkmd / fname)
            rel = png.relative_to(plots_root)
            title = f"{rel.parent.as_posix()}/{rel.stem}" if rel.parent != Path(".") else rel.stem
            _mkmd_notebook(
                title=title,
                caption=f"`{stem}.yml`: ESMValTool plot `{rel.as_posix()}`",
                experiment=experiment,
                nb_stem=stem,
                plot_fname=fname,
                mdfol=str(mkmd),
                repo_root=repo_root,
                nb_relpath=nb_relpath,
                kind="recipe",
            )
        return len(pngs)
    
    
    def tool_versions() -> dict[str, str]:
        """Installed versions of the tools a recipe run depends on (those not installed are left out)."""
        found = {}
        for dist in TOOL_DISTRIBUTIONS:
            try:
                found[dist] = metadata.version(dist)
            except metadata.PackageNotFoundError:
                pass
        return found
    
    
    def _write_marker(ofol: Path, stem: str, info: dict) -> None:
        info = {**info, "recipe": stem, "finished": datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M UTC")}
        marker_path(ofol, stem).write_text(json.dumps(info, indent=2) + "\n")
    
    
    def conda_module(environ=os.environ) -> str | None:
        """The conda/analysis3-* module this run used: MKFIGS_CONDA_MODULE (set by mkfigs.sh jobs), else LOADEDMODULES."""
        if environ.get("MKFIGS_CONDA_MODULE"):
            return environ["MKFIGS_CONDA_MODULE"]
        return next((m for m in environ.get("LOADEDMODULES", "").split(":") if m.startswith("conda/")), None)
    
    
    def _without_option(args: list[str], option: str) -> list[str]:
        out, skip = [], False
        for arg in args:
            if skip:
                skip = False
            elif arg == option:
                skip = True
            elif not arg.startswith(option + "="):
                out.append(arg)
        return out
    
    
    def reproduce_commands(stem: str, info: dict) -> str | None:
        """Shell commands that remake a recipe's plots outside mkfigs, from the run marker (None for older markers)."""
        rep = info.get("reproduce")
        if not rep:
            return None
        q = shlex.quote
        recipe = rep.get("recipe") or f"notebooks/recipes/{stem}.yml"
        lines = ["# On Gadi, in a clone of this repository made with --recurse-submodules:"]
        if rep.get("conda_module"):
            lines.append(f"module use /g/data/xp65/public/modules && module load {q(rep['conda_module'])}")
        lines += [
            "export PYTHONPATH=$PWD/external/access-model-mkfigs/src",
            f"WORK=/g/data/$PROJECT/$USER/{stem}   # any folder you can write to",
            "mkdir -p $WORK/esmvaltool_config",
        ]
        for name in rep.get("presets") or []:
            fname = name if name.endswith(".yml") else f"{name}.yml"
            lines.append(f"esmvaltool config copy {q(fname)} --target_file $WORK/esmvaltool_config/{q(fname)}   # ESMValCore data preset")
        if rep.get("input_root"):
            cmd = [f"python -m mkfigs.moppy_prepare {q(recipe)}", f"--input-root {q(rep['input_root'])}",
                   "--cache-dir $WORK/cmorised", "--output-config $WORK/esmvaltool_config/moppy-esmval-data.yml"]
            if rep.get("model_id"):
                cmd.append(f"--model-id {q(rep['model_id'])}")
            cmd += [f"--pattern {q(pat)}" for pat in rep.get("patterns") or []]
            extra = rep.get("moppy_args") or []
            cmd += [f"{a} {q(b)}" for a, b in zip(extra[::2], extra[1::2])]
            cmd.append("--user-config $WORK/moppy_user.yml")
            lines += ["", "# 1. CMORise the model output the recipe needs (ACCESS-MOPPy)", " \\\n    ".join(cmd)]
            lines += ["", "# 2. Run the recipe (use the .moppy-pinned.yml copy instead if step 1 wrote one next to it)"]
        else:
            lines += ["", "# Run the recipe"]
        esm = f"ESMVALTOOL_CONFIG_DIR=$WORK/esmvaltool_config esmvaltool run {q(recipe)} --output_dir $WORK/output"
        if rep.get("esmvaltool_args"):
            esm += " " + rep["esmvaltool_args"]
        lines += [esm, "", f"# Plots: $WORK/output/{Path(recipe).stem}_<YYYYMMDD_HHMMSS>/plots/"]
        return "\n".join(lines)
    
    
    def esmvalcore_presets_dir() -> Path | None:
        """ESMValCore's bundled config presets (``esmvalcore/config/configurations``), or None if it isn't installed."""
        spec = importlib.util.find_spec("esmvalcore")
        if spec is None or not spec.origin:
            return None
        return Path(spec.origin).parent / "config" / "configurations"
    
    
    def presets_from_env(environ=os.environ) -> list[str]:
        """``MKFIGS_ESMVALTOOL_PRESETS`` (names separated by spaces, commas or ':') as a list of preset names."""
        return [n for n in re.split(r"[\s,:]+", environ.get(PRESETS_ENV, "")) if n]
    
    
    TIMEOUT_ENV = "MKFIGS_RECIPE_TIMEOUT"
    
    
    def timeout_from_env(environ=os.environ) -> float | None:
        """``MKFIGS_RECIPE_TIMEOUT`` (minutes per ``esmvaltool run``) in seconds; None when unset, empty or 0."""
        raw = environ.get(TIMEOUT_ENV, "").strip()
        if not raw:
            return None
        try:
            minutes = float(raw)
        except ValueError:
            log.warning("Ignoring %s=%r: not a number of minutes", TIMEOUT_ENV, raw)
            return None
        return minutes * 60 if minutes > 0 else None
    
    
    def run_killable(cmd, cwd=None, env=None, timeout: float | None = None) -> subprocess.CompletedProcess:
        """subprocess.run, but on timeout kill the whole process group (esmvaltool's worker processes too).
    
        ESMValTool can hang instead of exiting when a diagnostic script crashes, which would hold the job until
        walltime. Returns returncode -SIGKILL and ``timed_out=True`` on timeout.
        """
        proc = subprocess.Popen(cmd, cwd=cwd, env=env, start_new_session=True)
        try:
            rc = proc.wait(timeout=timeout)
            timed_out = False
        except subprocess.TimeoutExpired:
            try:
                os.killpg(proc.pid, signal.SIGKILL)
            except ProcessLookupError:
                pass
            rc = proc.wait()
            timed_out = True
        done = subprocess.CompletedProcess(cmd, rc)
        done.timed_out = timed_out
        return done
    
    
    def _recipe_relpath(recipes_dir: Path, stem: str, repo_root: Path) -> str:
        """Path of the recipe relative to the repo root (for the git-history author lookup)."""
        try:
            return str((Path(recipes_dir) / f"{stem}.yml").resolve().relative_to(Path(repo_root).resolve()))
        except ValueError:
            return f"notebooks/recipes/{stem}.yml"
    
    
    def run_recipe(
        stem: str,
        ofol: Path | str,
        recipes_dir: Path | str,
        *,
        experiment: str,
        repo_root: Path | str,
        input_root: str = "",
        cache_dir: str = "",
        esmvaltool_args: str = "",
        model_id: str = "",
        patterns: list[str] | tuple[str, ...] = (),
        moppy_args: list[str] | tuple[str, ...] = (),
        base_config_dir: str = "",
        presets: list[str] | tuple[str, ...] = (),
        presets_dir: Path | str | None = None,
        timeout: float | None = None,
        runner=subprocess.run,
    ) -> bool:
        """Run one recipe end to end. Returns True if esmvaltool succeeded and produced at least one plot.
    
        A run marker is always written (also on failure) so pushit can report FAILED rather than NOT RUN.
        ``input_root`` is the ACCESS-MOPPy ``--input-root``; leave it empty to skip CMORisation (data already
        discoverable by ESMValTool). ``model_id`` is MOPPy's ``--model-id`` (e.g. ``ACCESS-OM3``) and ``patterns`` are
        ``COMPOUND_NAME:GLOB`` strings passed as ``--pattern`` (MOPPy's default globs follow the ACCESS-ESM1.6 archive
        layout). ``moppy_args`` are extra :mod:`mkfigs.moppy_prepare` options (``--supergrid``, ``--nominal-resolution``,
        ``--workers``, ``--user-config`` ...); without ``--user-config`` MOPPy's creator profile is ``<ofol>/moppy_user.yml``.
        ESMValTool runs with ``ESMVALTOOL_CONFIG_DIR=<ofol>/esmvaltool_config``, seeded with the ``*.yml`` files
        of ``base_config_dir`` when given and with the ESMValCore config ``presets`` named (e.g. ``data-hpc-nci``, from
        ``presets_dir``, default ESMValCore's own). ``timeout`` (seconds) caps ``esmvaltool run``: the run is killed and
        reported as failed (see :func:`run_killable`). ``runner`` is injectable for tests.
        """
        ofol, recipes_dir, repo_root = Path(ofol), Path(recipes_dir), Path(repo_root)
        mkmd = ofol / "mkmd"
        ofol.mkdir(parents=True, exist_ok=True)
        versions = tool_versions()
    
        src = recipes_dir / f"{stem}.yml"
        if not src.is_file():
            _write_marker(ofol, stem, {"returncode": 2, "error": f"recipe not found: {src}", "n_plots": 0, "versions": versions})
            return False
    
        work = ofol / "recipes"
        work.mkdir(exist_ok=True)
        recipe = work / f"{stem}.yml"
        shutil.copy2(src, recipe)
    
        config_dir = ofol / CONFIG_SUBDIR
        config_dir.mkdir(exist_ok=True)
        # ESMValCore reads every YAML here, so a data-source file left by an earlier run (another cache, another
        # seed directory) would silently add data roots. Start each recipe from an empty config set.
        for stale in config_dir.glob("*.yml"):
            stale.unlink()
        if base_config_dir:
            for cfg in sorted(Path(base_config_dir).expanduser().glob("*.yml")):
                if cfg.name != MOPPY_CONFIG_NAME:
                    shutil.copy2(cfg, config_dir / cfg.name)
        for name in presets:
            fname = name if name.endswith(".yml") else f"{name}.yml"
            where = Path(presets_dir) if presets_dir else esmvalcore_presets_dir()
            if where is None or not (where / fname).is_file():
                _write_marker(ofol, stem, {"returncode": 2, "error": f"ESMValCore config preset not found: {fname} (in {where})",
                                           "n_plots": 0, "versions": versions})
                return False
            shutil.copy2(where / fname, config_dir / fname)
        env = {**os.environ, "ESMVALTOOL_CONFIG_DIR": str(config_dir)}
    
        # A pinned copy left by an earlier run would otherwise be run instead of the current recipe.
        (work / f"{stem}.moppy-pinned.yml").unlink(missing_ok=True)
        if input_root:
            cache = str(Path(cache_dir or DEFAULT_MOPPY_CACHE).expanduser())
            cmd = [sys.executable, "-m", "mkfigs.moppy_prepare", str(recipe), "--input-root", input_root,
                   "--cache-dir", cache, "--output-config", str(config_dir / MOPPY_CONFIG_NAME)]
            if model_id:
                cmd += ["--model-id", model_id]
            for pat in patterns:
                cmd += ["--pattern", pat]
            cmd += list(moppy_args)
            if "--user-config" not in moppy_args:
                cmd += ["--user-config", str(ofol / MOPPY_USER_CONFIG_NAME)]
            prep = runner(cmd, cwd=str(ofol), env=env)
            if prep.returncode != 0:
                _write_marker(ofol, stem, {"returncode": prep.returncode, "error": "moppy-esmval-prepare failed",
                                           "n_plots": 0, "versions": versions})
                return False
            # Some ACCESS-MOPPy versions write a pinned copy of the recipe next to the one they were given.
            pinned = work / f"{stem}.moppy-pinned.yml"
            if pinned.is_file():
                recipe = pinned
    
        out = ofol / "esmvaltool" / stem
        out.mkdir(parents=True, exist_ok=True)
        esmvaltool_cmd = ["esmvaltool", "run", str(recipe), "--output_dir", str(out), *shlex.split(esmvaltool_args)]
        if timeout:
            result = (run_killable if runner is subprocess.run else runner)(
                esmvaltool_cmd, cwd=str(ofol), env=env, timeout=timeout)
        else:
            result = runner(esmvaltool_cmd, cwd=str(ofol), env=env)
        timed_out = bool(getattr(result, "timed_out", False))
    
        run_dir = find_run_dir(out)
        n_plots = 0
        if run_dir is not None:
            n_plots = collect_outputs(run_dir, mkmd, stem, experiment, repo_root, _recipe_relpath(recipes_dir, stem, repo_root))
            filled = find_filled_recipe(run_dir, Path(recipe).stem)
            shutil.copy2(filled or recipe, filled_recipe_path(ofol, stem))
    
        info = {
            "returncode": result.returncode,
            "run_dir": str(run_dir) if run_dir else None,
            "n_plots": n_plots,
            "recipe_used": recipe.name,
            "prepared": bool(input_root),
            "model_id": model_id or None,
            "config_dir": str(config_dir),
            "versions": versions,
            "reproduce": {  # what write_recipe_page needs to print the commands that remake these plots
                "recipe": _recipe_relpath(recipes_dir, stem, repo_root),
                "conda_module": conda_module(),
                "input_root": input_root or None,
                "model_id": model_id or None,
                "patterns": list(patterns),
                # the creator profile path is this run's own; the reproduce commands use their own
                "moppy_args": _without_option(list(moppy_args), "--user-config"),
                "esmvaltool_args": esmvaltool_args,
                "presets": list(presets),
            },
        }
        if timed_out:
            info["error"] = (f"esmvaltool run killed after {timeout / 60:g} min ({TIMEOUT_ENV}); it may have hung "
                             "after a diagnostic script failed: see the diagnostic logs in run/")
        elif result.returncode != 0:
            info["error"] = "esmvaltool run failed"
        elif run_dir is None:
            info["error"] = "no ESMValTool run directory with plots/ found"
        elif n_plots == 0:
            info["error"] = "esmvaltool succeeded but produced no PNG plots"
        _write_marker(ofol, stem, info)
        return result.returncode == 0 and n_plots > 0
    
    
    def extract_recipe_error(ofol: Path | str, stem: str, tail_lines: int = 40) -> str | None:
        """Short failure description for mkfigs_errors.log: the marker's error plus the end of ESMValTool's main log."""
        info = read_marker(ofol, stem)
        if info is None:
            return None
        parts = [str(info.get("error") or f"returncode {info.get('returncode')}")]
        run_dir = info.get("run_dir")
        if run_dir:
            log_file = Path(run_dir) / "run" / "main_log.txt"
            try:
                parts.append("\n".join(log_file.read_text().splitlines()[-tail_lines:]))
            except OSError:
                pass
        return "\n".join(parts)
    
    
    def write_recipe_page(ofol: Path | str, stem: str, dest_md: Path, ename: str) -> Path:
        """Write ``recipes/<stem>.md``: the recipe that produced the figures, with the tool versions that ran it.
    
        The body is wrapped in a Jinja raw block because the docs build renders Markdown through mkdocs-macros.
        """
        info = read_marker(ofol, stem) or {}
        filled = filled_recipe_path(ofol, stem)
        body = filled.read_text() if filled.is_file() else "# resolved recipe not found in the run output\n"
        fence = "`" * max(3, max((len(m) for m in re.findall(r"`+", body)), default=0) + 1)
        versions = info.get("versions") or {}
        rows = "".join(f"| {name} | {ver} |\n" for name, ver in sorted(versions.items()))
        lines = [
            "{% raw %}",
            f"# {stem}",
            "",
            f"ESMValTool recipe used to produce the figures for experiment **{ename}**."
            f" [Back to the figures](../{stem}.md).",
            "",
        ]
        if info.get("finished"):
            lines += [f"Run finished: {info['finished']}", ""]
        if rows:
            lines += ["| Tool | Version |", "|---|---|", rows.rstrip("\n"), ""]
        lines += [
            "This is ESMValTool's resolved copy of the recipe (`<recipe>_filled.yml`), i.e. what actually ran.",
            "",
            f"{fence}yaml",
            body.rstrip("\n"),
            fence,
            "",
        ]
        commands = reproduce_commands(stem, info)
        if commands:
            lines += [
                "## Reproduce these plots",
                "",
                "These commands repeat this run by hand: the same recipe, input data and settings, without mkfigs' page building."
                " Run them where the tool versions above are installed; the cached CMORised data can be reused for later runs.",
                "",
                "```bash",
                commands,
                "```",
                "",
                "To remake the whole website entry instead, set `ENAME`/`ESMDIR` in `notebooks/mkfigs.sh` and run "
                f"`qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE={info['reproduce'].get('conda_module') or '<module>'} mkfigs.sh`"
                " from `notebooks/`, then `python3 -m mkfigs.pushit`.",
                "",
            ]
        lines += ["{% endraw %}", ""]
        dest_md.parent.mkdir(parents=True, exist_ok=True)
        dest_md.write_text("\n".join(lines))
        return dest_md
    ````

??? abstract "access-model-mkfigs/src/mkfigs/run.py (diff, +71 −5)"

    --kind notebooks|recipes|all; recipe loop; separate recipe error log.

    ```diff
    --- a/src/mkfigs/run.py
    +++ b/src/mkfigs/run.py
    @@ -27,6 +27,8 @@ import sys
     from pathlib import Path
     
     from . import get_mkfigs_version
    +from . import recipes as recipes_mod
    +from ._paths import find_repo_root
     
     
     def _extract_notebook_error(rendered_path: Path) -> str | None:
    @@ -97,6 +99,16 @@ def parse_args() -> argparse.Namespace:
                  "Default '' preserves the original access-om3-paper-1 layout, "
                  "where notebooks sit directly in <wfolder>/notebooks/.",
         )
    +    p.add_argument(
    +        "--kind", choices=("notebooks", "recipes", "all"), default="notebooks",
    +        help="What to run: the MKFIGS_NOTEBOOKS notebooks (default, the original behaviour), the "
    +             "MKFIGS_RECIPES ESMValTool recipes, or both. Recipes usually need a different conda "
    +             "environment from the notebooks, so they are normally run as a separate job.",
    +    )
    +    p.add_argument(
    +        "--recipes-subdir", default="recipes",
    +        help="Subdirectory of the notebooks directory holding the <recipe>.yml files (default 'recipes').",
    +    )
         return p.parse_args()
     
     
    @@ -166,14 +178,26 @@ def main() -> None:
         log.info("Log file              : %s", mdfol / "mkfigs_run.log")
         log.info("access-model-mkfigs   : %s", mkfigs_ver)
     
    -    notebooks_env = os.environ.get("MKFIGS_NOTEBOOKS", "")
    -    if not notebooks_env:
    -        log.error("MKFIGS_NOTEBOOKS env var not set — run via mkfigs.sh")
    -        sys.exit(1)
    -    notebooks = [n for n in notebooks_env.split(":") if n]
    +    notebooks: list[str] = []
    +    if args.kind in ("notebooks", "all"):
    +        notebooks_env = os.environ.get("MKFIGS_NOTEBOOKS", "")
    +        if not notebooks_env:
    +            log.error("MKFIGS_NOTEBOOKS env var not set — run via mkfigs.sh")
    +            sys.exit(1)
    +        notebooks = [n for n in notebooks_env.split(":") if n]
    +
    +    recipes: list[str] = []
    +    if args.kind in ("recipes", "all"):
    +        recipes_env = os.environ.get("MKFIGS_RECIPES", "")
    +        if not recipes_env:
    +            log.error("MKFIGS_RECIPES env var not set — run via mkfigs.sh")
    +            sys.exit(1)
    +        recipes = [r for r in recipes_env.split(":") if r]
     
         succeeded: list[str] = []
         failed:    list[str] = []
    +    succeeded_recipes: list[str] = []
    +    failed_recipes:    list[str] = []
     
         log.info("")
         log.info("Running %d notebooks ...", len(notebooks))
    @@ -188,6 +212,34 @@ def main() -> None:
                 failed.append(nb)
                 log.error("FAILED %s", nb)
     
    +    if recipes:
    +        log.info("")
    +        log.info("Running %d ESMValTool recipes ...", len(recipes))
    +        log.info("")
    +    for rec in recipes:
    +        log.info("START  recipe %s", rec)
    +        ok = recipes_mod.run_recipe(
    +            rec, ofol, notebooks_dir / args.recipes_subdir,
    +            experiment=ename,
    +            repo_root=find_repo_root(notebooks_dir),
    +            input_root=os.environ.get("MKFIGS_MOPPY_INPUT_ROOT", ""),
    +            cache_dir=os.environ.get("MKFIGS_MOPPY_CACHE", ""),
    +            esmvaltool_args=os.environ.get("MKFIGS_ESMVALTOOL_ARGS", ""),
    +            model_id=os.environ.get("MKFIGS_MOPPY_MODEL_ID", ""),
    +            # Newline- or ';'-separated COMPOUND_NAME:GLOB entries (globs contain ':'-free paths, so ':' can't separate).
    +            patterns=[p.strip() for p in re.split(r"[;\n]", os.environ.get("MKFIGS_MOPPY_PATTERNS", "")) if p.strip()],
    +            moppy_args=recipes_mod.moppy_args_from_env(),
    +            base_config_dir=os.environ.get("MKFIGS_ESMVALTOOL_CONFIG_DIR", ""),
    +            presets=recipes_mod.presets_from_env(),
    +            timeout=recipes_mod.timeout_from_env(),
    +        )
    +        if ok:
    +            succeeded_recipes.append(rec)
    +            log.info("OK     recipe %s", rec)
    +        else:
    +            failed_recipes.append(rec)
    +            log.error("FAILED recipe %s", rec)
    +
         log.info("")
         log.info("=" * 56)
         log.info("Run complete")
    @@ -212,6 +264,20 @@ def main() -> None:
                     ef.write("\n")
             log.error("Error details: %s", errors_log)
     
    +    if succeeded_recipes:
    +        log.info("Succeeded recipes (%d): %s", len(succeeded_recipes), ", ".join(succeeded_recipes))
    +    if failed_recipes:
    +        log.error("FAILED    recipes (%d): %s", len(failed_recipes), ", ".join(failed_recipes))
    +        # A separate file, so a recipes job never overwrites the notebooks job's mkfigs_errors.log.
    +        recipe_errors_log = mdfol / "mkfigs_recipe_errors.log"
    +        with open(recipe_errors_log, "w") as ef:
    +            for rec in failed_recipes:
    +                ef.write(f"{'=' * 56}\nFAILED recipe: {rec}\n{'=' * 56}\n")
    +                error = recipes_mod.extract_recipe_error(ofol, rec)
    +                ef.write(error + "\n" if error else "(no run marker found)\n")
    +                ef.write("\n")
    +        log.error("Error details: %s", recipe_errors_log)
    +
         _print_next_steps(Path(args.wfolder), notebooks_dir)
     
     def _print_next_steps(wfolder: Path, notebooks_dir: Path) -> None:
    ```

??? abstract "access-model-mkfigs/src/mkfigs/pushit.py (diff, +208 −25)"

    Recipes in their own landing-table section and nav group, optionally split by RECIPE_CATEGORIES; a Recipe page per recipe.

    ```diff
    --- a/src/mkfigs/pushit.py
    +++ b/src/mkfigs/pushit.py
    @@ -49,6 +49,7 @@ from . import get_mkfigs_version
     from .configdoc import figshare_upload_and_rewrite, assign_pngs_to_notebooks, FigshareUploader, _md5
     from .configdoc import get_docs_url, get_model_name, get_repo_slug, get_repo_url
     from ._paths import find_notebooks_dir, find_repo_root
    +from .recipes import read_marker, recipe_stems_in, write_recipe_page
     
     
     def _check_nci_environment() -> None:
    @@ -96,6 +97,16 @@ def _load_notebook_categories(notebooks_dir: Path) -> dict[str, list[int | str]]
         return getattr(_load_mkfigs_issues_module(notebooks_dir), "CATEGORIES", {})
     
     
    +def _load_recipe_categories(notebooks_dir: Path) -> dict[str, list[str]]:
    +    """Load the optional RECIPE_CATEGORIES dict from mkfigs_issues.py.
    +
    +    RECIPE_CATEGORIES maps a sub-heading to a list of recipe stems; the recipes are then shown as
    +    sub-groups of the "ESMValTool recipes" section and nav group (see group_recipes_by_category).
    +    Returns {} when absent: one flat recipe table and nav group.
    +    """
    +    return getattr(_load_mkfigs_issues_module(notebooks_dir), "RECIPE_CATEGORIES", {})
    +
    +
     def _load_categories_source(notebooks_dir: Path) -> str | None:
         """Load the optional CATEGORIES_SOURCE URL (the issue the categories mirror)."""
         return getattr(_load_mkfigs_issues_module(notebooks_dir), "CATEGORIES_SOURCE", None)
    @@ -145,6 +156,26 @@ def parse_mkfigs_sh() -> tuple[str, str, list[str]]:
         return ename, esmdir, notebooks
     
     
    +def parse_recipes_from_mkfigs_sh() -> list[str]:
    +    """Parse the optional ESMValTool recipe list (a ``recipes=( ... )`` bash array) from mkfigs.sh.
    +
    +    Returns [] when mkfigs.sh has no such array, so paper repos that only use notebooks are unaffected.
    +    Whole-line comments are dropped first, so a commented-out example array is never picked up.
    +    """
    +    sh = HERE / "mkfigs.sh"
    +    if not sh.exists():
    +        return []
    +    text = "\n".join(l for l in sh.read_text().splitlines() if not l.lstrip().startswith("#"))
    +    arr_match = re.search(r'\brecipes=\(\s*(.*?)\s*\)', text, re.DOTALL)
    +    recipes: list[str] = []
    +    if arr_match:
    +        for raw in arr_match.group(1).splitlines():
    +            stripped = raw.split("#", 1)[0].strip()
    +            if stripped:
    +                recipes.append(stripped)
    +    return recipes
    +
    +
     HERE = _find_notebooks_dir()
     REPO = _find_repo_root(HERE)
     DOCS_PAGES = REPO / "documentation" / "docs" / "pages"
    @@ -207,9 +238,42 @@ def get_authors_md() -> str:
     # ---------------------------------------------------------------------------
     STATUS_HEADER = "Notebook run success?"
     UNCATEGORISED = "Miscellaneous"
    +# ESMValTool recipes always get their own landing-page section and nav group, after the notebook categories.
    +RECIPES_CATEGORY = "ESMValTool recipes"
    +# Sub-group for recipes not listed in RECIPE_CATEGORIES (only used when RECIPE_CATEGORIES is set).
    +UNCATEGORISED_RECIPES = "Other recipes"
     _ISSUE_NUM_RE = re.compile(r"/issues/(\d+)\b")
     
     
    +def group_recipes_by_category(
    +    recipes: list[str], recipe_categories: dict[str, list[str]] | None, warn: bool = False,
    +) -> dict[str, list[str]]:
    +    """Split recipe stems into RECIPE_CATEGORIES sub-groups, in RECIPE_CATEGORIES order.
    +
    +    A recipe listed under several sub-groups goes to the first. Recipes not listed go to
    +    UNCATEGORISED_RECIPES (last). Empty sub-groups are dropped. Each group is sorted alphabetically.
    +    Without RECIPE_CATEGORIES everything is one group named RECIPES_CATEGORY.
    +    """
    +    ordered = sorted(recipes, key=str.lower)
    +    if not recipe_categories:
    +        return {RECIPES_CATEGORY: ordered} if ordered else {}
    +    grouped: dict[str, list[str]] = {}
    +    placed: set[str] = set()
    +    for cat, stems in recipe_categories.items():
    +        members = [r for r in ordered if r in stems and r not in placed]
    +        placed.update(members)
    +        if members:
    +            grouped.setdefault(cat, []).extend(members)
    +    rest = [r for r in ordered if r not in placed]
    +    if rest:
    +        if warn:
    +            for r in rest:
    +                print(f"[categories] recipe '{r}' is not in RECIPE_CATEGORIES; listing it under "
    +                      f"'{UNCATEGORISED_RECIPES}'.")
    +        grouped.setdefault(UNCATEGORISED_RECIPES, []).extend(rest)
    +    return grouped
    +
    +
     def _category_anchor(name: str) -> str:
         """Stable HTML id for a category heading, e.g. 'cat-sea-ice'."""
         return "cat-" + re.sub(r"[^a-z0-9]+", "-", name.lower()).strip("-")
    @@ -303,15 +367,24 @@ def build_run_summary(
         mkfigs_version: str = "unknown",
         existing_run_versions: dict[str, str] | None = None,
         categories: dict[str, list[int | str]] | None = None,
    +    recipes: set[str] | frozenset[str] | None = None,
    +    recipe_categories: dict[str, list[str]] | None = None,
     ) -> tuple[str, str]:
         """Return (plain_text_summary, markdown_summary).
     
         ``notebook_issues`` maps notebook stem to a Markdown GitHub issue string,
         loaded from mkfigs_issues.py in the paper repo's notebooks/ directory.
     
    +    ``recipes`` (optional) is the subset of the stems that are ESMValTool recipes
    +    rather than notebooks: they get a "Recipe" link instead of a "Notebook" link,
    +    and the table headings say "Notebook / recipe" so both kinds read naturally.
    +
         ``categories`` (optional, CATEGORIES in mkfigs_issues.py) splits the
         notebook table into one sub-headed table per category, with a "Jump to"
         line above them. Without it a single flat table is produced.
    +    Recipes always get their own "ESMValTool recipes" section (before Miscellaneous), whatever CATEGORIES says.
    +    ``recipe_categories`` (optional, RECIPE_CATEGORIES in mkfigs_issues.py) splits that section into one
    +    sub-headed table per sub-group (see group_recipes_by_category).
     
         Both contain the same information; the markdown version uses a table and
         bullet list suitable for embedding directly in pages/index.md.
    @@ -338,7 +411,11 @@ def build_run_summary(
         # Markdown for pages/index.md — links included in table so no separate Sections block needed
         exp_dir = f"experiments/{ename}"
     
    +    recipes = recipes or frozenset()
    +
         def _links(nb: str) -> str:
    +        if nb in recipes:
    +            return f"[Figures]({exp_dir}/{nb}.md)&nbsp;·&nbsp;[Recipe]({exp_dir}/recipes/{nb}.md)"
             return f"[Figures]({exp_dir}/{nb}.md)&nbsp;·&nbsp;[Notebook]({exp_dir}/notebooks/{nb}/)"  # nbsp: keep on one line
     
         _status: dict[str, tuple[str, str]] = {}   # nb -> (run-success label, links cell)
    @@ -355,9 +432,12 @@ def build_run_summary(
             # Stable order across runs: alphabetical within each category table.
             all_nbs = sorted(all_nbs, key=str.lower)
     
    +    first_col = "Notebook / recipe" if recipes else "Notebook"
    +    run_col = "Notebooks / recipes run" if recipes else "Notebooks run"
    +
         def _table(nbs: list[str]) -> str:
             return (
    -            f"| Notebook | {STATUS_HEADER} | GitHub Issue(s) | Links |\n"
    +            f"| {first_col} | {STATUS_HEADER} | GitHub Issue(s) | Links |\n"
                 "|---|---|---|---|\n"
                 + "\n".join(
                     f"| `{nb}` | {_status[nb][0]} | {notebook_issues.get(nb, '')} | {_status[nb][1]} |"
    @@ -366,15 +446,33 @@ def build_run_summary(
                 + "\n"
             )
     
    -    if categories:
    -        grouped = group_notebooks_by_category(all_nbs, notebook_issues, categories, warn=True)
    +    if categories or recipes:
    +        notebooks_only = [nb for nb in all_nbs if nb not in recipes]
    +        grouped = (group_notebooks_by_category(notebooks_only, notebook_issues, categories, warn=True)
    +                   if categories else ({"Notebooks": notebooks_only} if notebooks_only else {}))
    +        recipe_rows = [nb for nb in all_nbs if nb in recipes]
    +        recipe_groups = group_recipes_by_category(recipe_rows, recipe_categories, warn=True)
    +        if recipe_rows:  # own section, before Miscellaneous
    +            misc = grouped.pop(UNCATEGORISED, None)
    +            grouped[RECIPES_CATEGORY] = recipe_rows
    +            if misc:
    +                grouped[UNCATEGORISED] = misc
             jump = " · ".join(
                 f"[{cat}](#{_category_anchor(cat)}) ({len(nbs)})" for cat, nbs in grouped.items()
             )
    -        # Avoid Jinja's "{#" comment delimiter when mkdocs-macros renders this Markdown.
    +
    +        def _section(cat: str, nbs: list[str]) -> str:
    +            # Avoid Jinja's "{#" comment delimiter when mkdocs-macros renders this Markdown.
    +            head = f'### {cat} {{id="{_category_anchor(cat)}"}}\n\n'
    +            if cat == RECIPES_CATEGORY and recipe_categories:
    +                return head + "\n".join(
    +                    f'#### {sub} {{id="{_category_anchor("recipes " + sub)}"}}\n\n{_table(rs)}'
    +                    for sub, rs in recipe_groups.items()
    +                )
    +            return head + _table(nbs)
    +
             tables_md = f"**Jump to:** {jump}\n\n" + "\n".join(
    -            f'### {cat} {{id="{_category_anchor(cat)}"}}\n\n{_table(nbs)}'
    -            for cat, nbs in grouped.items()
    +            _section(cat, nbs) for cat, nbs in grouped.items()
             )
         else:
             tables_md = _table(all_nbs)
    @@ -391,7 +489,7 @@ def build_run_summary(
             _by_run.setdefault((_nb_times[nb], _nb_versions[nb]), []).append(nb)
     
         run_time_md = (
    -        "| Date | Notebooks run | access-model-mkfigs |\n"
    +        f"| Date | {run_col} | access-model-mkfigs |\n"
             "|---|---|---|\n"
             + "\n".join(
                 f"| {t} | {', '.join('`' + nb + '`' for nb in sorted(nbs))} | {_mkfigs_version_md(v)} |"
    @@ -445,10 +543,14 @@ def _get_figshare_article_url(mdfol: Path, ename: str) -> str | None:
     # Figshare upload (per notebook: PNGs + rendered notebook with outputs)
     # ---------------------------------------------------------------------------
     def upload_figshare_for_notebook(
    -    mdfol: Path, ofol: Path, ename: str, nb_name: str, token: str
    +    mdfol: Path, ofol: Path, ename: str, nb_name: str, token: str, is_recipe: bool = False
     ) -> str | None:
         """Upload PNGs + the fully rendered notebook for *nb_name* to figshare.
     
    +    With *is_recipe* (an ESMValTool recipe, see mkfigs.recipes) there is no rendered
    +    notebook: only the PNGs are uploaded and the markdown rewritten. The recipe itself is
    +    small and goes into git as recipes/<stem>.md, so it needs no Figshare URL.
    +
         The rendered notebook (``<ofol>/<nb_name>_rendered.ipynb``) still has its
         cell outputs intact at this point — mkfigs_run.py strips them only for the
         copy that goes back into ``notebooks/``.  We upload the output-bearing copy
    @@ -459,7 +561,9 @@ def upload_figshare_for_notebook(
         """
     
         nb_path = ofol / f"{nb_name}_rendered.ipynb"
    -    if not nb_path.exists():
    +    if is_recipe:
    +        nb_path = None
    +    elif not nb_path.exists():
             print(f"[figshare] WARNING: rendered notebook not found: {nb_path}")
             nb_path = None
     
    @@ -680,6 +784,11 @@ def check_figshare_upload_mode(ename: str) -> None:
             for nb in ok_nbs
             if (experiment_docs_dir / f"{nb}.md").exists()
         ]
    +    # ESMValTool recipes: their figure page and the recipe page (no notebook to add).
    +    for rec in parse_recipes_from_mkfigs_sh():
    +        for page in (experiment_docs_dir / f"{rec}.md", experiment_docs_dir / "recipes" / f"{rec}.md"):
    +            if page.exists():
    +                added_files.append(str(page.relative_to(REPO)))
         added_files += [
             urls_json_rel,
             "documentation/docs/pages/index.md",
    @@ -989,15 +1098,32 @@ def _ensure_mkdocs_jupyter_plugin(data: dict) -> dict:
         return data
     
     
    -def update_mkdocs_nav(ename: str, ok_nbs: list[str], dry_run: bool = False) -> None:
    +def update_mkdocs_nav(
    +    ename: str,
    +    ok_nbs: list[str],
    +    dry_run: bool = False,
    +    recipes: set[str] | frozenset[str] | None = None,
    +    recipe_categories: dict[str, list[str]] | None = None,
    +) -> None:
         """Add (or replace) per-notebook nav entries in mkdocs.yml.
     
    +    Recipes go in one "ESMValTool recipes" group after the notebooks, split into
    +    RECIPE_CATEGORIES sub-groups when ``recipe_categories`` is given.
    +
         Each notebook gets a top-level entry with summary and notebook sub-pages:
     
           - <nb>:
               - summary:  pages/experiments/<ename>/<nb>.md
               - notebook: pages/experiments/<ename>/notebooks/<nb>.ipynb
     
    +    Stems in *recipes* (ESMValTool recipes) go in one "ESMValTool recipes" group after the notebooks, each with a
    +    "Recipe" page instead of a notebook:
    +
    +      - ESMValTool recipes:
    +          - <recipe>:
    +              - Summary: pages/experiments/<ename>/<recipe>.md
    +              - Recipe:  pages/experiments/<ename>/recipes/<recipe>.md
    +
         Notebooks live in a notebooks/ subdirectory to avoid URL collision with the
         summary .md (the hide_pages hook strips 'pages/' from both, so same-stem
         files in the same dir would resolve to the same URL).
    @@ -1014,30 +1140,43 @@ def update_mkdocs_nav(ename: str, ok_nbs: list[str], dry_run: bool = False) -> N
     
         exp_dir = f"pages/experiments/{ename}"
     
    -    def _is_experiment_entry(item: dict) -> bool:
    -        """True for old-style 'Evaluation figures' sections and per-notebook entries
    -        whose paths belong to pages/experiments/ (from any previous run)."""
    -        if "Evaluation figures" in item:
    -            return True
    -        for sub in item.values():
    -            if isinstance(sub, list):
    -                for child in sub:
    -                    if isinstance(child, dict):
    -                        for v in child.values():
    -                            if isinstance(v, str) and "pages/experiments/" in v:
    -                                return True
    +    def _points_into_experiments(node) -> bool:
    +        if isinstance(node, str):
    +            return "pages/experiments/" in node
    +        if isinstance(node, dict):
    +            return any(_points_into_experiments(v) for v in node.values())
    +        if isinstance(node, list):
    +            return any(_points_into_experiments(v) for v in node)
             return False
     
    +    def _is_experiment_entry(item: dict) -> bool:
    +        """True for old-style 'Evaluation figures' sections, per-notebook entries and the recipes group,
    +        i.e. anything whose paths belong to pages/experiments/ (from any previous run)."""
    +        return "Evaluation figures" in item or _points_into_experiments(item)
    +
         # Remove all stale experiment entries (old structure or previous runs).
         nav = [item for item in nav if not (isinstance(item, dict) and _is_experiment_entry(item))]
     
         # Append fresh entries in a stable, alphabetical order.
    +    recipes = recipes or frozenset()
    +    def _recipe_entry(nb: str) -> dict:
    +        return {nb: [{"Summary": f"{exp_dir}/{nb}.md"}, {"Recipe": f"{exp_dir}/recipes/{nb}.md"}]}
    +
    +    recipe_groups = group_recipes_by_category([nb for nb in ok_nbs if nb in recipes], recipe_categories)
    +    if recipe_categories:
    +        recipe_entries = [{sub: [_recipe_entry(nb) for nb in rs]} for sub, rs in recipe_groups.items()]
    +    else:
    +        recipe_entries = [_recipe_entry(nb) for rs in recipe_groups.values() for nb in rs]
         for nb in sorted(ok_nbs, key=str.lower):
    +        if nb in recipes:
    +            continue
             nav.append({nb: [
                 {"Summary":  f"{exp_dir}/{nb}.md"},
                 {"Notebook": f"{exp_dir}/notebooks/{nb}.ipynb"},
             ]})
     
    +    if recipe_entries:  # one group after the notebooks, like the landing-page section
    +        nav.append({RECIPES_CATEGORY: recipe_entries})
         data["nav"] = nav
     
         if dry_run:
    @@ -1197,6 +1336,7 @@ def main() -> None:
         print(f"[mkfigs] Running from commit {mkfigs_version}")
     
         ename, esmdir, notebooks = parse_mkfigs_sh()
    +    recipes = parse_recipes_from_mkfigs_sh()
         if args.ename:
             ename = args.ename
     
    @@ -1234,6 +1374,7 @@ def main() -> None:
                 if p.name[: -len(suffix)] not in notebooks
             )
             notebooks = notebooks + extra
    +        recipes = recipes + [r for r in recipe_stems_in(ofol) if r not in recipes]
     
         # -----------------------------------------------------------------------
         # Load previously committed URLs before notebook classification so we can
    @@ -1267,7 +1408,10 @@ def main() -> None:
         # prefix of another's (e.g. "MLD" matching "MLD_max"'s PNGs too),
         # which can make a notebook look OK/FAILED based on the wrong PNGs.
         # Resolve ownership once, up front, against the full notebook list.
    -    pngs_by_notebook = assign_pngs_to_notebooks(mdfol, notebooks) if mdfol.exists() else {}
    +    # ESMValTool recipes (mkfigs.recipes) own PNGs by the same <stem>_NN.png rule, so they join the same pool.
    +    pngs_by_notebook = assign_pngs_to_notebooks(mdfol, notebooks + recipes) if mdfol.exists() else {}
    +    recipe_set: set[str] = set(recipes)
    +    recipe_pages_dir = experiment_docs_dir / "recipes"
     
         for nb in notebooks:
             rendered = ofol / f"{nb}_rendered.ipynb"
    @@ -1290,16 +1434,41 @@ def main() -> None:
                 status = f"OK      ({n_png} PNG{'s' if n_png != 1 else ''})"
             print(f"  {status:<34}  {nb}")
     
    +    # ESMValTool recipes: the run marker written by mkfigs.recipes stands in for <nb>_rendered.ipynb.
    +    for rec in recipes:
    +        info = read_marker(ofol, rec)
    +        pngs = pngs_by_notebook.get(rec, [])
    +        if info is None and (recipe_pages_dir / f"{rec}.md").exists():
    +            prev_committed_nbs.append(rec)  # committed by an earlier render; nothing to restore for recipes
    +            status = "PREV COMMITTED (recipe)"
    +        elif info is None:
    +            not_run_nbs.append(rec)
    +            status = "NOT RUN (recipe)"
    +        elif info.get("returncode") != 0 or not pngs:
    +            failed_nbs.append(rec)
    +            status = "FAILED  (recipe run failed or no PNGs)"
    +        else:
    +            ok_nbs.append(rec)
    +            status = f"OK      ({len(pngs)} PNG{'s' if len(pngs) != 1 else ''}, recipe)"
    +        print(f"  {status:<34}  {rec}")
    +
         # Any previously committed notebooks absent from this run's notebook list.
         _seen = {*ok_nbs, *failed_nbs, *not_run_nbs, *prev_committed_nbs}
         for nb in existing_urls:
             if nb not in _seen:
                 prev_committed_nbs.append(nb)
    +    # ... and previously committed recipes (their pages are already in the docs tree).
    +    if recipe_pages_dir.is_dir():
    +        for page in sorted(recipe_pages_dir.glob("*.md")):
    +            if page.stem not in _seen:
    +                prev_committed_nbs.append(page.stem)
    +                recipe_set.add(page.stem)
     
         print()
         notebook_issues = _load_notebook_issues(HERE)
         notebook_categories = _load_notebook_categories(HERE)
         categories_source = _load_categories_source(HERE)
    +    recipe_categories = _load_recipe_categories(HERE)
         plain_summary, md_summary = build_run_summary(
             ename, esmdir, ok_nbs, failed_nbs, not_run_nbs, prev_committed_nbs, run_time,
             notebook_issues=notebook_issues,
    @@ -1307,6 +1476,8 @@ def main() -> None:
             existing_run_times=existing_run_times,
             mkfigs_version=mkfigs_version,
             existing_run_versions=existing_run_versions,
    +        recipes=recipe_set,
    +        recipe_categories=recipe_categories,
         )
         print(plain_summary)
     
    @@ -1331,7 +1502,7 @@ def main() -> None:
             token = resolve_figshare_token()
             if token:
                 for nb in ok_nbs:
    -                nb_url = upload_figshare_for_notebook(mdfol, ofol, ename, nb, token)
    +                nb_url = upload_figshare_for_notebook(mdfol, ofol, ename, nb, token, is_recipe=nb in recipe_set)
                     if nb_url:
                         notebook_urls[nb] = nb_url
             else:
    @@ -1382,6 +1553,8 @@ def main() -> None:
     
         # Rendered notebooks back into notebooks/ (outputs stripped here, after Figshare upload)
         for nb in ok_nbs:
    +        if nb in recipe_set:
    +            continue  # recipes have no notebook to copy back (their page is written below)
             src = ofol / f"{nb}_rendered.ipynb"
             dst = HERE / f"{nb}.ipynb"
             print(f"  {src.name}  ->  {dst.relative_to(REPO)}  (outputs stripped for git)")
    @@ -1406,6 +1579,15 @@ def main() -> None:
             else:
                 print(f"  WARNING: {src_md} not found — skipping")
     
    +    # ESMValTool recipe pages: embed the (resolved) recipe that produced each recipe's figures.
    +    for nb in ok_nbs:
    +        if nb not in recipe_set:
    +            continue
    +        dst_page = recipe_pages_dir / f"{nb}.md"
    +        print(f"  {nb}_recipe_filled.yml  ->  {dst_page.relative_to(REPO)}")
    +        if not args.dry_run:
    +            write_recipe_page(ofol, nb, dst_page, ename)
    +
         # Write notebooks_urls.json so .readthedocs.yaml can download notebooks
         # at build time (notebooks are too large to commit to git).
         if args.dry_run:
    @@ -1441,7 +1623,8 @@ def main() -> None:
         # -----------------------------------------------------------------------
         # Update mkdocs.yml nav (+ ensure mkdocs-jupyter plugin present)
         # -----------------------------------------------------------------------
    -    update_mkdocs_nav(ename=ename, ok_nbs=all_nav_nbs, dry_run=args.dry_run)
    +    update_mkdocs_nav(ename=ename, ok_nbs=all_nav_nbs, dry_run=args.dry_run, recipes=recipe_set,
    +                      recipe_categories=recipe_categories)
     
         # -----------------------------------------------------------------------
         # Next step: publish on Figshare, then get git commands
    ```

??? abstract "access-model-mkfigs/src/mkfigs/configdoc.py (diff, +18 −6)"

    Figure markdown can say it came from a recipe rather than a notebook.

    ```diff
    --- a/src/mkfigs/configdoc.py
    +++ b/src/mkfigs/configdoc.py
    @@ -707,10 +707,11 @@ class FigshareUploader:
                 return results
     
             ofol = os.path.dirname(os.path.normpath(self.mdfol))
    +        # Recipes (mkfigs.recipes) own PNGs the same way notebooks do; their run marker stands in for the rendered notebook.
             all_notebook_names = [
    -            f[: -len("_rendered.ipynb")]
    +            f[: -len("_rendered.ipynb")] if f.endswith("_rendered.ipynb") else f[: -len("_recipe_run.json")]
                 for f in os.listdir(ofol)
    -            if f.endswith("_rendered.ipynb")
    +            if f.endswith(("_rendered.ipynb", "_recipe_run.json"))
             ] if os.path.isdir(ofol) else [nb_name]
             if nb_name not in all_notebook_names:
                 all_notebook_names.append(nb_name)
    @@ -1040,12 +1041,15 @@ class MkmdWriter:
     # ---------------------------------------------------------------------------
     
     
    -def _mkmd_notebook(title, caption, experiment, nb_stem, plot_fname, mdfol, repo_root, table="", model_name=None, nb_relpath=None):
    +def _mkmd_notebook(title, caption, experiment, nb_stem, plot_fname, mdfol, repo_root, table="", model_name=None, nb_relpath=None, kind="notebook"):
         """Write (or append) a figure/table entry to ``<mdfol>/<nb_stem>.md``.
     
         Each notebook gets its own markdown file named after the notebook stem
         (e.g. ``SST.md``, ``MLD.md``).  The header is written only on the first
         call; subsequent calls from the same notebook append just the new section.
    +
    +    ``kind="recipe"`` (see mkfigs.recipes) words the header for an ESMValTool recipe
    +    and links the recipe page instead of the rendered notebook.
         """
         model_name = model_name or get_model_name()
         try:
    @@ -1097,15 +1101,23 @@ def _mkmd_notebook(title, caption, experiment, nb_stem, plot_fname, mdfol, repo_
         if first_write:
             _authors = get_notebook_authors(nb_stem, repo_root, nb_relpath=nb_relpath)
             _authors_str = ", ".join(_authors) if _authors else "unknown"
    +        if kind == "recipe":
    +            produced = f"ESMValTool recipe `{nb_stem}.yml`"
    +            coauthors = "Co-authors for this recipe"
    +            view_link = f"[View recipe](recipes/{nb_stem}.md)"
    +        else:
    +            produced = f"notebook `{nb_stem}.ipynb`"
    +            coauthors = "Co-authors for this notebook"
    +            view_link = f"[View rendered notebook](notebooks/{nb_stem}.ipynb)"
             header = [
                 "<!-- auto-generated by mkfigs_configdoc.py – do not edit manually -->\n",
                 f"# {nb_stem}\n",
                 " \n",
                 (
                     f"Evaluation figures from {model_name} experiment **{experiment}**"
    -                f" produced by notebook `{nb_stem}.ipynb`."
    -                f" Co-authors for this notebook (via Git history): {_authors_str}."
    -                f" [View rendered notebook](notebooks/{nb_stem}.ipynb)\n"
    +                f" produced by {produced}."
    +                f" {coauthors} (via Git history): {_authors_str}."
    +                f" {view_link}\n"
                 ),
                 " \n",
             ]
    ```

??? abstract "access-model-mkfigs/tests/test_moppy_prepare.py (new file, +129 −0)"

    New. Runs against the real access_moppy (skipped where MOPPy < 1.8).

    ```python
    # Copyright 2024 ACCESS-NRI and contributors. See the top-level COPYRIGHT file for details.
    # SPDX-License-Identifier: Apache-2.0
    
    """Tests for mkfigs.moppy_prepare (the ACCESS-MOPPy workarounds). Need access_moppy >= 1.8; skipped otherwise."""
    
    from __future__ import annotations
    
    import json
    
    import pytest
    import yaml
    
    from mkfigs import moppy_prepare
    
    if not moppy_prepare.moppy_version_ok():  # older MOPPy prompts for creator details on import
        pytest.skip("needs access-moppy >= 1.8", allow_module_level=True)
    access_moppy = pytest.importorskip("access_moppy")
    from access_moppy import _config, ocean_supergrid  # noqa: E402
    from access_moppy import vocabulary_processors as vp  # noqa: E402
    from access_moppy.esmval import cli_commands  # noqa: E402
    from access_moppy.esmval.orchestrator import CMORiseOrchestrator  # noqa: E402
    
    
    @pytest.fixture
    def restore_moppy(monkeypatch):
        """Undo every in-memory patch after the test (monkeypatch restores what it saw first)."""
        monkeypatch.setattr(CMORiseOrchestrator, "prepare_recipe", CMORiseOrchestrator.prepare_recipe)
        monkeypatch.setattr(ocean_supergrid.Supergrid, "get_supergrid_path", ocean_supergrid.Supergrid.get_supergrid_path)
        monkeypatch.setattr(_config, "CONFIG_PATH", _config.CONFIG_PATH)
        monkeypatch.setattr(vp, "_CMIP6_TEMP_SOURCE_OVERRIDES", dict(vp._CMIP6_TEMP_SOURCE_OVERRIDES))
        monkeypatch.setattr(cli_commands, "_ACCESS_DATASETS", cli_commands._ACCESS_DATASETS)
        from access_moppy import base
        original = getattr(base, "_mkfigs_original_preprocess", base._preprocess_open_mfdataset)
        monkeypatch.setattr(base, "_preprocess_open_mfdataset", original)
        monkeypatch.delattr(base, "_mkfigs_original_preprocess", raising=False)
    
    
    def test_main_applies_workarounds_and_passes_the_rest_to_moppy(tmp_path, monkeypatch, restore_moppy):
        hgrid = tmp_path / "ocean_hgrid.nc"
        hgrid.write_bytes(b"")
        seen = {}
    
        def fake_main_prepare(argv):
            seen["argv"] = argv
            seen["allowed"] = []
            monkeypatch.setattr("access_moppy.esmval.orchestrator.RecipeReader",
                                lambda path, allowed_datasets=None: seen["allowed"].append(allowed_datasets)
                                or type("R", (), {"tasks": []})())
            CMORiseOrchestrator(input_root=tmp_path, cache_dir=tmp_path / "c", model_id="ACCESS-OM3").prepare_recipe("r.yml")
            return 0
    
        monkeypatch.setattr(cli_commands, "main_prepare", fake_main_prepare)
        user_cfg = tmp_path / "out" / "moppy_user.yml"
        rc = moppy_prepare.main([
            "r.yml", "--input-root", "/in", "--model-id", "ACCESS-OM3", "--pattern", "Omon.mlotst:x*.nc",
            "--supergrid", str(hgrid), "--nominal-resolution", "25 km", "--user-config", str(user_cfg),
        ])
        assert rc == 0
        # mkfigs options are consumed; MOPPy's own are passed through unchanged
        assert seen["argv"] == ["r.yml", "--input-root", "/in", "--model-id", "ACCESS-OM3", "--pattern", "Omon.mlotst:x*.nc"]
        assert seen["allowed"] == [frozenset({"ACCESS-OM3"})]
        entry = vp._CMIP6_TEMP_SOURCE_OVERRIDES["ACCESS-OM3"]
        assert entry["source_id"] == "ACCESS-OM3"
        assert entry["model_component"]["ocean"]["native_nominal_resolution"] == "25 km"
        assert "ACCESS-OM2" not in json.dumps(entry["model_component"]) and "MOM5" not in json.dumps(entry["model_component"])
        assert entry["model_component"]["atmos"]["description"] == "none"
        assert ocean_supergrid.Supergrid.get_supergrid_path(None, "25 km") == str(hgrid)
        assert _config.CONFIG_PATH == str(user_cfg)
        assert yaml.safe_load(user_cfg.read_text()) == moppy_prepare.DEFAULT_CREATOR
    
    
    def test_existing_user_config_is_kept(tmp_path, restore_moppy):
        cfg = tmp_path / "me.yml"
        cfg.write_text("creator_name: Me\n")
        moppy_prepare.use_user_config(str(cfg))
        assert cfg.read_text() == "creator_name: Me\n" and _config.CONFIG_PATH == str(cfg)
    
    
    def test_register_source_id_leaves_models_already_in_the_cv_alone(restore_moppy):
        assert moppy_prepare.register_source_id("ACCESS-ESM1-5", "ACCESS-OM2") is False
        assert "ACCESS-ESM1-5" not in vp._CMIP6_TEMP_SOURCE_OVERRIDES
    
    
    def test_missing_supergrid_is_an_error(tmp_path, restore_moppy):
        with pytest.raises(SystemExit):
            moppy_prepare.use_supergrid(str(tmp_path / "nope.nc"))
    
    
    @pytest.mark.parametrize("argv, expected", [
        (["r.yml", "--model-id", "ACCESS-OM3"], "ACCESS-OM3"),
        (["r.yml", "--model-id=ACCESS-OM3"], "ACCESS-OM3"),
        (["r.yml"], ""),
    ])
    def test_model_id_parsing(argv, expected):
        assert moppy_prepare._model_id(argv) == expected
    
    
    def test_recipe_versions_are_not_pinned(tmp_path, restore_moppy):
        recipe = tmp_path / "r.yml"
        recipe.write_text(yaml.safe_dump({
            "datasets": [{"dataset": "ACCESS-OM3", "project": "CMIP6", "exp": "omip2"},
                         {"dataset": "ACCESS-ESM1-5", "project": "CMIP6", "exp": "historical"}],
            "diagnostics": {"d": {"variables": {"tos": {"mip": "Omon"}}}},
        }))
        moppy_prepare.allow_datasets(frozenset({"ACCESS-OM3"}))
        # No pinned copy: published ACCESS-ESM1-5 keeps its own version, and ESMValCore picks the newest cached one.
        assert cli_commands._pin_recipe_dataset_version(recipe, version="v20261112") == (None, 0)
    
    
    def test_z_level_axes_are_read_as_native_mom6_axes(restore_moppy, monkeypatch):
        import numpy as np
        import xarray as xr
        from access_moppy import base
    
        moppy_prepare.support_z_level_output()
        moppy_prepare.support_z_level_output()   # idempotent: wraps once
        assert base._mkfigs_original_preprocess is not moppy_prepare._z_level_preprocess
    
        ds = xr.Dataset(
            {"so": (("time", "z_l", "yh", "xh"), np.zeros((1, 2, 1, 1)))},
            coords={"time": [0.0], "z_l": ("z_l", [1.0, 3.0], {"edges": "z_i"}), "z_i": [0.0, 2.0, 5.0],
                    "yh": [0.0], "xh": [0.0]},
        )
        out = base._preprocess_open_mfdataset(ds, required_vars=frozenset({"so", "zl", "zi"}))
        assert out["so"].dims == ("time", "zl", "yh", "xh")
        assert list(out["zi"].values) == [0.0, 2.0, 5.0] and out["zl"].attrs["edges"] == "zi"
        # native MOM6 output is passed through unchanged
        native = ds.rename({"z_l": "zl", "z_i": "zi"})
        assert base._preprocess_open_mfdataset(native, required_vars=frozenset({"so"}))["so"].dims == ("time", "zl", "yh", "xh")
    ```

??? abstract "access-model-mkfigs/tests/test_recipes.py (new file, +614 −0)"

    New. Runner, layout, pushit and run.main tests with a fake ESMValTool.

    `````python
    """ESMValTool recipe support (mkfigs.recipes) and its pushit / run integration.
    
    All ESMValTool and ACCESS-MOPPy calls are replaced by a fake runner that writes the output layout
    mkfigs expects from ESMValTool (<recipe stem>_<timestamp>/plots/... and run/<recipe stem>_filled.yml), as seen in a
    real ESMValTool 2.15 run on Gadi (2026-11-11).
    """
    from __future__ import annotations
    
    import json
    import sys
    from pathlib import Path
    from types import SimpleNamespace
    from unittest.mock import patch
    
    import pytest
    import yaml
    
    from mkfigs import pushit, recipes, run as run_mod
    
    from .conftest import make_png, make_rendered_notebook
    
    ENAME = "test_experiment_01"
    
    
    def make_run_dir(out: Path, name: str = "demo", stamp: str = "20261110_100000", plots=None) -> Path:
        """Write a fake ESMValTool run directory under *out* and return it."""
        run_dir = out / f"{name}_{stamp}"
        (run_dir / "plots").mkdir(parents=True, exist_ok=True)
        for rel in plots if plots is not None else ["diagA/taskA/fig1.png", "diagA/taskB/fig2.png"]:
            png = run_dir / "plots" / rel
            png.parent.mkdir(parents=True, exist_ok=True)
            make_png(png)
        (run_dir / "run").mkdir(parents=True, exist_ok=True)
        (run_dir / "run" / f"{name}_filled.yml").write_text("documentation:\n  title: Demo\n")
        return run_dir
    
    
    PREPARE = [sys.executable, "-m", "mkfigs.moppy_prepare"]
    
    
    def fake_runner(esmvaltool_rc=0, prepare_rc=0, write_pinned=False, plots=None, calls=None):
        """A stand-in for subprocess.run that fakes mkfigs.moppy_prepare and esmvaltool run."""
        def runner(cmd, cwd=None, **kwargs):
            if calls is not None:
                calls.append(list(cmd))
            if cmd[:3] == PREPARE:
                if write_pinned:
                    Path(cmd[3]).with_name(Path(cmd[3]).stem + ".moppy-pinned.yml").write_text("pinned: true\n")
                return SimpleNamespace(returncode=prepare_rc)
            assert cmd[0] == "esmvaltool" and cmd[1] == "run"
            out = Path(cmd[cmd.index("--output_dir") + 1])
            if esmvaltool_rc == 0 or plots is not None:
                make_run_dir(out, plots=plots)
            return SimpleNamespace(returncode=esmvaltool_rc)
        return runner
    
    
    @pytest.fixture
    def recipe_repo(tmp_path):
        """A repo root with notebooks/recipes/demo.yml and an output folder for ENAME."""
        repo = tmp_path / "paper-repo"
        (repo / "notebooks" / "recipes").mkdir(parents=True)
        (repo / "notebooks" / "recipes" / "demo.yml").write_text("documentation:\n  title: Demo\n")
        ofol = repo / "notebooks" / f"mkfigs_output_{ENAME}"
        ofol.mkdir()
        return repo, ofol
    
    
    # ---------------------------------------------------------------------------
    # recipes module
    # ---------------------------------------------------------------------------
    
    def test_find_run_dir_returns_newest_and_ignores_dirs_without_plots(tmp_path):
        make_run_dir(tmp_path, stamp="20261109_090000")
        newest = make_run_dir(tmp_path, stamp="20261110_100000")
        (tmp_path / "recipe_demo_20261111_000000").mkdir()  # an aborted run: no plots/
        assert recipes.find_run_dir(tmp_path) == newest
        assert recipes.find_run_dir(tmp_path / "missing") is None
    
    
    def test_collect_outputs_names_pngs_after_the_stem_and_writes_markdown(tmp_path):
        run_dir = make_run_dir(tmp_path / "esmvaltool")
        mkmd = tmp_path / "mkmd"
        n = recipes.collect_outputs(run_dir, mkmd, "demo", ENAME, tmp_path)
        assert n == 2
        assert sorted(p.name for p in mkmd.glob("*.png")) == ["demo_01.png", "demo_02.png"]
        md = (mkmd / "demo.md").read_text()
        assert "produced by ESMValTool recipe `demo.yml`" in md
        assert "[View recipe](recipes/demo.md)" in md
        assert f"(/assets/experiments/{ENAME}/demo_01.png)" in md
        assert "diagA/taskA/fig1" in md  # title from the plot's path under plots/
    
    
    def test_collect_outputs_replaces_an_earlier_runs_output(tmp_path):
        mkmd = tmp_path / "mkmd"
        recipes.collect_outputs(make_run_dir(tmp_path / "a"), mkmd, "demo", ENAME, tmp_path)
        make_png(mkmd / "demo_max_01.png")  # another recipe whose stem starts with "demo": must survive
        n = recipes.collect_outputs(make_run_dir(tmp_path / "b", plots=["only.png"]), mkmd, "demo", ENAME, tmp_path)
        assert n == 1
        assert sorted(p.name for p in mkmd.glob("*.png")) == ["demo_01.png", "demo_max_01.png"]
        assert (mkmd / "demo.md").read_text().count("## ") == 1
    
    
    def test_run_recipe_success_prepares_runs_collects_and_writes_marker(recipe_repo):
        repo, ofol = recipe_repo
        calls: list[list[str]] = []
        ok = recipes.run_recipe(
            "demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
            input_root="/g/data/x/archive", cache_dir=str(repo / "cache"),
            esmvaltool_args="--max-parallel-tasks 4", runner=fake_runner(calls=calls),
        )
        assert ok is True
        assert calls[0][:3] == PREPARE
        assert calls[0][calls[0].index("--input-root") + 1] == "/g/data/x/archive"
        assert calls[0][calls[0].index("--cache-dir") + 1] == str(repo / "cache")
        assert calls[1][:2] == ["esmvaltool", "run"]
        assert calls[1][-2:] == ["--max-parallel-tasks", "4"]
        marker = json.loads(recipes.marker_path(ofol, "demo").read_text())
        assert marker["returncode"] == 0 and marker["n_plots"] == 2 and marker["prepared"] is True
        assert "error" not in marker
        assert recipes.filled_recipe_path(ofol, "demo").read_text().startswith("documentation:")
        assert (ofol / "mkmd" / "demo_01.png").exists() and (ofol / "mkmd" / "demo.md").exists()
        assert (repo / "notebooks" / "recipes" / "demo.yml").read_text() == "documentation:\n  title: Demo\n"  # source untouched
    
    
    def test_run_recipe_runs_the_moppy_pinned_copy_when_prepare_writes_one(recipe_repo):
        repo, ofol = recipe_repo
        calls: list[list[str]] = []
        recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                           input_root="/in", cache_dir="/c", runner=fake_runner(write_pinned=True, calls=calls))
        assert calls[1][2].endswith("demo.moppy-pinned.yml")
        assert json.loads(recipes.marker_path(ofol, "demo").read_text())["recipe_used"] == "demo.moppy-pinned.yml"
    
    
    def test_run_recipe_skips_cmorisation_without_an_input_root(recipe_repo):
        repo, ofol = recipe_repo
        calls: list[list[str]] = []
        assert recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  runner=fake_runner(calls=calls))
        assert [c[0] for c in calls] == ["esmvaltool"]
        assert json.loads(recipes.marker_path(ofol, "demo").read_text())["prepared"] is False
    
    
    @pytest.mark.parametrize(
        "runner_kwargs, expected_error",
        [
            ({"prepare_rc": 3}, "moppy-esmval-prepare failed"),
            ({"esmvaltool_rc": 1}, "esmvaltool run failed"),
            ({"plots": []}, "produced no PNG plots"),
        ],
    )
    def test_run_recipe_failures_still_write_a_marker(recipe_repo, runner_kwargs, expected_error):
        repo, ofol = recipe_repo
        ok = recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                input_root="/in", cache_dir="/c", runner=fake_runner(**runner_kwargs))
        assert ok is False
        marker = recipes.read_marker(ofol, "demo")
        assert marker is not None and expected_error in marker["error"]
        assert expected_error in recipes.extract_recipe_error(ofol, "demo")
    
    
    def test_run_recipe_missing_source_is_a_failure_not_a_crash(recipe_repo):
        repo, ofol = recipe_repo
        assert recipes.run_recipe("nope", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  runner=fake_runner()) is False
        assert "recipe not found" in recipes.read_marker(ofol, "nope")["error"]
    
    
    def test_write_recipe_page_embeds_the_recipe_safely(tmp_path):
        ofol = tmp_path / f"mkfigs_output_{ENAME}"
        ofol.mkdir()
        recipes.marker_path(ofol, "demo").write_text(json.dumps(
            {"returncode": 0, "finished": "2026-11-10 10:00 UTC", "versions": {"ESMValTool": "2.15.0"}}))
        # a recipe containing a code fence and Jinja-looking text must neither break the page nor be rendered
        recipes.filled_recipe_path(ofol, "demo").write_text("note: |\n  ```\n  {{ not_jinja }}\n  ```\n")
        page = recipes.write_recipe_page(ofol, "demo", tmp_path / "pages" / "recipes" / "demo.md", ENAME)
        text = page.read_text()
        assert text.startswith("{% raw %}") and text.rstrip().endswith("{% endraw %}")
        assert "````yaml" in text and "| ESMValTool | 2.15.0 |" in text
        assert "[Back to the figures](../demo.md)" in text
        body = text.split("````yaml\n")[1].split("\n````")[0]
        assert yaml.safe_load(body)["note"].startswith("```")
    
    
    # ---------------------------------------------------------------------------
    # pushit integration
    # ---------------------------------------------------------------------------
    
    def _seed_recipe(ofol: Path, stem="demo", returncode=0, n_png=1):
        mdfol = ofol / "mkmd"
        mdfol.mkdir(parents=True, exist_ok=True)
        recipes.marker_path(ofol, stem).write_text(json.dumps(
            {"returncode": returncode, "n_plots": n_png, "versions": {"ESMValTool": "2.15.0"}}))
        recipes.filled_recipe_path(ofol, stem).write_text("documentation:\n  title: Demo\n")
        for i in range(1, n_png + 1):
            make_png(mdfol / f"{stem}_{i:02d}.png")
        if n_png:
            (mdfol / f"{stem}.md").write_text(f"# {stem}\n\n![fig](/assets/experiments/{ENAME}/{stem}_01.png)\n")
    
    
    def _set_mkfigs_sh(repo: Path, recipes_array="demo"):
        sh = repo / "notebooks" / "mkfigs.sh"
        sh.write_text(
            "#!/bin/bash\n"
            "# an example, not a real list:\n"
            "#   recipes=( commented_out )\n"
            f"ENAME={ENAME}\n"
            "ESMDIR=/g/data/tm70/fake/${ENAME}/datastore.json\n"
            "array=(\n  SST\n)\n"
            f"recipes=(\n  {recipes_array}  # trailing comment\n)\n"
        )
    
    
    def _run_pushit(args):
        with patch.object(sys, "argv", ["mkfigs-pushit", *args]), patch.object(pushit.subprocess, "run") as mock_run:
            pushit.main()
        return mock_run
    
    
    def test_parse_recipes_ignores_commented_examples_and_missing_arrays(patch_repo_paths):
        _set_mkfigs_sh(patch_repo_paths, recipes_array="demo\n  other")
        assert pushit.parse_recipes_from_mkfigs_sh() == ["demo", "other"]
        (patch_repo_paths / "notebooks" / "mkfigs.sh").write_text("ENAME=x\nESMDIR=y\narray=(\n SST\n)\n")
        assert pushit.parse_recipes_from_mkfigs_sh() == []
    
    
    def test_pushit_publishes_a_recipe_alongside_a_notebook(patch_repo_paths):
        repo = patch_repo_paths
        _set_mkfigs_sh(repo)
        ofol = repo / "notebooks" / f"mkfigs_output_{ENAME}"
        (ofol / "mkmd").mkdir(parents=True)
        make_rendered_notebook(ofol / "SST_rendered.ipynb")
        make_png(ofol / "mkmd" / "SST_01.png")
        (ofol / "mkmd" / "SST.md").write_text("# SST\n")
        _seed_recipe(ofol)
    
        mock_run = _run_pushit(["--skip-figshare"])
    
        docs = repo / "documentation" / "docs" / "pages"
        exp = docs / "experiments" / ENAME
        assert (exp / "demo.md").exists() and (exp / "SST.md").exists()
        assert "documentation:" in (exp / "recipes" / "demo.md").read_text()
        assert not (repo / "notebooks" / "demo.ipynb").exists()
        # nbconvert --clear-output ran for the notebook only
        cleared = [c.args[0][-1] for c in mock_run.call_args_list if "--clear-output" in c.args[0]]
        assert cleared == [str(repo / "notebooks" / "SST.ipynb")]
    
        nav = {next(iter(i)): i[next(iter(i))] for i in yaml.safe_load((repo / "documentation" / "mkdocs.yml").read_text())["nav"][1:]}
        assert "demo" not in nav  # recipes are grouped, after the notebooks
        assert list(nav)[-1] == pushit.RECIPES_CATEGORY
        assert nav[pushit.RECIPES_CATEGORY] == [{"demo": [{"Summary": f"pages/experiments/{ENAME}/demo.md"},
                                                          {"Recipe": f"pages/experiments/{ENAME}/recipes/demo.md"}]}]
        assert nav["SST"][1] == {"Notebook": f"pages/experiments/{ENAME}/notebooks/SST.ipynb"}
    
        index = (docs / "index.md").read_text()
        assert "| Notebook / recipe |" in index and "Notebooks / recipes run" in index
        assert f"[Recipe](experiments/{ENAME}/recipes/demo.md)" in index
        assert f"[Notebook](experiments/{ENAME}/notebooks/SST/)" in index
        # the recipe has its own section, not the notebooks' table
        recipes_at = index.index(f"### {pushit.RECIPES_CATEGORY}")
        assert index.index("`SST`") < recipes_at < index.index("`demo`")
    
    
    def test_pushit_reports_failed_and_not_run_recipes(patch_repo_paths, capsys):
        repo = patch_repo_paths
        _set_mkfigs_sh(repo, recipes_array="bad\n  never_ran")
        ofol = repo / "notebooks" / f"mkfigs_output_{ENAME}"
        _seed_recipe(ofol, "bad", returncode=1, n_png=0)
        _run_pushit(["--dry-run"])
        out = capsys.readouterr().out
        assert "FAILED" in out and "bad" in out
        assert "NOT RUN (recipe)" in out and "never_ran" in out
    
    
    def test_recipe_without_a_new_run_stays_listed_as_previously_committed(patch_repo_paths):
        repo = patch_repo_paths
        _set_mkfigs_sh(repo)
        ofol = repo / "notebooks" / f"mkfigs_output_{ENAME}"
        (ofol / "mkmd").mkdir(parents=True)
        make_rendered_notebook(ofol / "SST_rendered.ipynb")
        make_png(ofol / "mkmd" / "SST_01.png")
        (ofol / "mkmd" / "SST.md").write_text("# SST\n")
        page = repo / "documentation" / "docs" / "pages" / "experiments" / ENAME / "recipes" / "demo.md"
        page.parent.mkdir(parents=True)
        page.write_text("committed earlier\n")
    
        _run_pushit(["--skip-figshare"])
    
        index = (repo / "documentation" / "docs" / "pages" / "index.md").read_text()
        assert "| `demo` | ✅ Yes (earlier run) |" in index
        nav = yaml.safe_load((repo / "documentation" / "mkdocs.yml").read_text())["nav"]
        assert {pushit.RECIPES_CATEGORY: [{"demo": [{"Summary": f"pages/experiments/{ENAME}/demo.md"},
                                                    {"Recipe": f"pages/experiments/{ENAME}/recipes/demo.md"}]}]} in nav
        assert page.read_text() == "committed earlier\n"
    
    
    def test_notebook_only_repos_are_unchanged(patch_repo_paths):
        """No recipes array: same table headings and nav as before this feature."""
        repo = patch_repo_paths
        ofol = repo / "notebooks" / f"mkfigs_output_{ENAME}"
        (ofol / "mkmd").mkdir(parents=True)
        make_rendered_notebook(ofol / "SST_rendered.ipynb")
        make_png(ofol / "mkmd" / "SST_01.png")
        (ofol / "mkmd" / "SST.md").write_text("# SST\n")
        _run_pushit(["--skip-figshare"])
        index = (repo / "documentation" / "docs" / "pages" / "index.md").read_text()
        assert "| Notebook |" in index and "Notebook / recipe" not in index and "[Recipe]" not in index
    
    
    # ---------------------------------------------------------------------------
    # run.main
    # ---------------------------------------------------------------------------
    
    def _run_main(monkeypatch, wfolder: Path, *extra):
        monkeypatch.setattr("sys.argv", ["mkfigs-run", "--ename", "exp1", "--esmdir", "/fake/esm.json",
                                         "--wfolder", str(wfolder), *extra])
        run_mod.main()
    
    
    def test_main_kind_recipes_runs_only_recipes_and_logs_failures_separately(tmp_path, monkeypatch):
        wfolder = tmp_path / "paper-repo"
        (wfolder / "notebooks").mkdir(parents=True)
        monkeypatch.setenv("MKFIGS_RECIPES", "good:bad")
        monkeypatch.setenv("MKFIGS_MOPPY_INPUT_ROOT", "/g/data/x/archive")
        monkeypatch.delenv("MKFIGS_NOTEBOOKS", raising=False)
        seen = []
    
        def fake_run_recipe(stem, ofol, recipes_dir, **kwargs):
            seen.append((stem, Path(recipes_dir).name, kwargs["input_root"], kwargs["experiment"]))
            if stem == "bad":
                recipes.marker_path(ofol, stem).write_text(json.dumps({"returncode": 1, "error": "esmvaltool run failed"}))
            return stem == "good"
    
        monkeypatch.setattr(run_mod.recipes_mod, "run_recipe", fake_run_recipe)
        monkeypatch.setattr(run_mod, "run_notebook", lambda *a, **k: pytest.fail("notebooks must not run"))
        _run_main(monkeypatch, wfolder, "--kind", "recipes")
    
        assert seen == [("good", "recipes", "/g/data/x/archive", "exp1"), ("bad", "recipes", "/g/data/x/archive", "exp1")]
        mdfol = wfolder / "notebooks" / "mkfigs_output_exp1" / "mkmd"
        errors = (mdfol / "mkfigs_recipe_errors.log").read_text()
        assert "FAILED recipe: bad" in errors and "esmvaltool run failed" in errors and "good" not in errors
        assert not (mdfol / "mkfigs_errors.log").exists()
    
    
    def test_main_kind_recipes_requires_the_recipes_env_var(tmp_path, monkeypatch):
        wfolder = tmp_path / "paper-repo"
        (wfolder / "notebooks").mkdir(parents=True)
        monkeypatch.delenv("MKFIGS_RECIPES", raising=False)
        with pytest.raises(SystemExit):
            _run_main(monkeypatch, wfolder, "--kind", "recipes")
    
    
    def test_main_default_kind_never_touches_recipes(tmp_path, monkeypatch):
        wfolder = tmp_path / "paper-repo"
        (wfolder / "notebooks").mkdir(parents=True)
        monkeypatch.setenv("MKFIGS_NOTEBOOKS", "SST")
        monkeypatch.setenv("MKFIGS_RECIPES", "demo")
        monkeypatch.setattr(run_mod, "run_notebook", lambda *a, **k: True)
        monkeypatch.setattr(run_mod.recipes_mod, "run_recipe", lambda *a, **k: pytest.fail("recipes must not run"))
        _run_main(monkeypatch, wfolder)
    
    
    def test_run_recipe_passes_moppy_model_patterns_and_keeps_config_out_of_home(recipe_repo, tmp_path):
        repo, ofol = recipe_repo
        base = tmp_path / "base_cfg"
        base.mkdir()
        (base / "data.yml").write_text("rootpath: {}\n")
        envs: list[dict] = []
        calls: list[list[str]] = []
        inner = fake_runner(calls=calls)
    
        def runner(cmd, cwd=None, env=None, **kwargs):
            envs.append(env)
            return inner(cmd, cwd=cwd, **kwargs)
    
        assert recipes.run_recipe(
            "demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
            input_root="/in", cache_dir="/c", model_id="ACCESS-OM3",
            patterns=["Omon.tos:output*/access-om3.mom6.2d.tos.1mon.mean.*.nc"], base_config_dir=str(base), runner=runner,
        )
        cfg_dir = ofol / recipes.CONFIG_SUBDIR
        prep = calls[0]
        assert prep[prep.index("--output-config") + 1] == str(cfg_dir / recipes.MOPPY_CONFIG_NAME)
        assert prep[prep.index("--model-id") + 1] == "ACCESS-OM3"
        assert prep[prep.index("--pattern") + 1] == "Omon.tos:output*/access-om3.mom6.2d.tos.1mon.mean.*.nc"
        assert all(e["ESMVALTOOL_CONFIG_DIR"] == str(cfg_dir) for e in envs)
        assert (cfg_dir / "data.yml").read_text() == "rootpath: {}\n"
        # MOPPy's creator profile defaults to the output folder: not $HOME, not the ESMValTool config dir
        assert prep[prep.index("--user-config") + 1] == str(ofol / recipes.MOPPY_USER_CONFIG_NAME)
    
    
    def test_run_recipe_passes_extra_moppy_args_and_a_given_user_config(recipe_repo):
        repo, ofol = recipe_repo
        calls: list[list[str]] = []
        extra = ["--supergrid", "/g/hgrid.nc", "--user-config", "/cfg/me.yml"]
        assert recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  input_root="/in", moppy_args=extra, runner=fake_runner(calls=calls))
        assert calls[0][-len(extra):] == extra
        assert calls[0].count("--user-config") == 1
    
    
    def test_moppy_args_from_env():
        env = {"MKFIGS_MOPPY_SUPERGRID": "/g/hgrid.nc", "MKFIGS_MOPPY_NOMINAL_RES": "25 km", "MKFIGS_MOPPY_WORKERS": " ",
               "MKFIGS_MOPPY_DATASETS": "ACCESS-OM3:ACCESS-OM3-alt"}
        assert recipes.moppy_args_from_env(env) == [
            "--supergrid", "/g/hgrid.nc", "--nominal-resolution", "25 km",
            "--dataset", "ACCESS-OM3", "--dataset", "ACCESS-OM3-alt",
        ]
        assert recipes.moppy_args_from_env({}) == []
    
    
    def test_find_run_dir_real_layout(tmp_path):
        """Run dirs are <recipe stem>_<YYYYMMDD_HHMMSS>; newest wins; dirs without plots/ or a timestamp are ignored."""
        assert recipes.find_run_dir(tmp_path / "missing") is None
        make_run_dir(tmp_path, name="om3_mlotst_basic", stamp="20261110_220040")
        newest = make_run_dir(tmp_path, name="om3_mlotst_basic", stamp="20261110_235640")
        (tmp_path / "om3_mlotst_basic_20261111_000000").mkdir()  # failed run: no plots/
        (tmp_path / "notes" / "plots").mkdir(parents=True)        # not a run dir
        assert recipes.find_run_dir(tmp_path) == newest
    
    
    def test_find_filled_recipe(tmp_path):
        run_dir = make_run_dir(tmp_path, name="om3_mlotst_basic")
        assert recipes.find_filled_recipe(run_dir, "om3_mlotst_basic") == run_dir / "run" / "om3_mlotst_basic_filled.yml"
        assert recipes.find_filled_recipe(run_dir, "other") == run_dir / "run" / "om3_mlotst_basic_filled.yml"
        (run_dir / "run" / "om3_mlotst_basic_filled.yml").unlink()
        assert recipes.find_filled_recipe(run_dir, "om3_mlotst_basic") is None
    
    
    def test_run_recipe_drops_config_files_left_by_an_earlier_run(recipe_repo):
        repo, ofol = recipe_repo
        cfg_dir = ofol / recipes.CONFIG_SUBDIR
        cfg_dir.mkdir(parents=True)
        (cfg_dir / "old-cache-data.yml").write_text("rootpath: {CMIP6: /old/cache}\n")
        assert recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  runner=fake_runner())
        assert not (cfg_dir / "old-cache-data.yml").exists()
    
    
    def test_recipes_get_their_own_section_before_miscellaneous():
        from datetime import datetime, timezone
        cats = {"Sea ice": ["SeaIce_area"], "Miscellaneous": ["odd"]}
        _, md = pushit.build_run_summary(
            "e", "/x.json", ok_nbs=["SeaIce_area", "zz_recipe", "odd", "a_recipe"], failed_nbs=[], not_run_nbs=[],
            prev_committed_nbs=[], run_time=datetime(2026, 1, 1, tzinfo=timezone.utc), categories=cats,
            recipes={"zz_recipe", "a_recipe"})
        heads = [line for line in md.splitlines() if line.startswith("### ")]
        assert [h.split(" {")[0] for h in heads] == ["### Sea ice", f"### {pushit.RECIPES_CATEGORY}", "### Miscellaneous"]
        section = md[md.index(f"### {pushit.RECIPES_CATEGORY}"):md.index("### Miscellaneous")]
        assert section.index("`a_recipe`") < section.index("`zz_recipe`") and "`odd`" not in section
    
    
    def test_nav_regrouping_replaces_an_old_recipes_group(patch_repo_paths):
        pushit.update_mkdocs_nav(ENAME, ["SST", "demo"], recipes={"demo"})
        pushit.update_mkdocs_nav(ENAME, ["SST", "demo"], recipes={"demo"})
        nav = yaml.safe_load((patch_repo_paths / "documentation" / "mkdocs.yml").read_text())["nav"]
        assert sum(1 for i in nav if pushit.RECIPES_CATEGORY in i) == 1
    
    
    def test_recipe_page_ends_with_commands_that_reproduce_the_run(recipe_repo, monkeypatch, tmp_path):
        repo, ofol = recipe_repo
        monkeypatch.setenv("MKFIGS_CONDA_MODULE", "conda/analysis3-26.11")
        assert recipes.run_recipe(
            "demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo, input_root="/g/data/ol01/run",
            model_id="ACCESS-OM3", patterns=["Omon.mlotst:output[0-9]*/x.*.nc"], esmvaltool_args="--max-parallel-tasks 4",
            moppy_args=["--supergrid", "/g/grid.nc", "--nominal-resolution", "25 km", "--user-config", "/mine.yml"],
            runner=fake_runner())
        text = recipes.write_recipe_page(ofol, "demo", tmp_path / "recipes" / "demo.md", ENAME).read_text()
        section = text.split("## Reproduce these plots")[1]
        assert "module load conda/analysis3-26.11" in section
        assert "python -m mkfigs.moppy_prepare notebooks/recipes/demo.yml" in section
        assert "--pattern 'Omon.mlotst:output[0-9]*/x.*.nc'" in section and "--nominal-resolution '25 km'" in section
        assert "/mine.yml" not in section and "--user-config $WORK/moppy_user.yml" in section
        assert "esmvaltool run notebooks/recipes/demo.yml --output_dir $WORK/output --max-parallel-tasks 4" in section
        assert text.rstrip().endswith("{% endraw %}")
    
    
    def test_recipe_page_without_reproduce_info_has_no_commands(tmp_path):
        ofol = tmp_path / "o"
        ofol.mkdir()
        recipes.marker_path(ofol, "demo").write_text(json.dumps({"returncode": 0}))
        assert "Reproduce" not in recipes.write_recipe_page(ofol, "demo", tmp_path / "d.md", ENAME).read_text()
    
    
    def test_conda_module_detection():
        assert recipes.conda_module({"MKFIGS_CONDA_MODULE": "conda/a"}) == "conda/a"
        assert recipes.conda_module({"LOADEDMODULES": "pbs:conda/analysis3-26.11:git"}) == "conda/analysis3-26.11"
        assert recipes.conda_module({}) is None
    
    
    def test_run_recipe_copies_named_esmvalcore_presets(recipe_repo, tmp_path):
        repo, ofol = recipe_repo
        presets = tmp_path / "configurations"
        presets.mkdir()
        (presets / "data-hpc-nci.yml").write_text("projects: {OBS6: {}}\n")
        (presets / "data-local.yml").write_text("projects: {}\n")
        assert recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  presets=["data-hpc-nci"], presets_dir=presets, runner=fake_runner())
        cfg = ofol / recipes.CONFIG_SUBDIR
        assert (cfg / "data-hpc-nci.yml").read_text() == "projects: {OBS6: {}}\n"
        assert not (cfg / "data-local.yml").exists()
        assert recipes.read_marker(ofol, "demo")["reproduce"]["presets"] == ["data-hpc-nci"]
    
    
    def test_run_recipe_unknown_preset_is_a_failure_not_a_crash(recipe_repo, tmp_path):
        repo, ofol = recipe_repo
        calls = []
        assert not recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                      presets=["data-hpc-nowhere"], presets_dir=tmp_path, runner=fake_runner(calls=calls))
        assert "data-hpc-nowhere.yml" in recipes.read_marker(ofol, "demo")["error"]
        assert calls == []
    
    
    def test_presets_from_env():
        assert recipes.presets_from_env({}) == []
        assert recipes.presets_from_env({"MKFIGS_ESMVALTOOL_PRESETS": "data-hpc-nci, extra:more"}) == [
            "data-hpc-nci", "extra", "more"]
    
    
    def test_reproduce_commands_copy_the_presets():
        text = recipes.reproduce_commands("demo", {"reproduce": {"recipe": "notebooks/recipes/demo.yml",
                                                                "presets": ["data-hpc-nci"]}})
        assert ("esmvaltool config copy data-hpc-nci.yml --target_file $WORK/esmvaltool_config/data-hpc-nci.yml"
                in text)
    
    
    def test_group_recipes_by_category_orders_and_collects_unlisted(capsys):
        cats = {"B group": ["r2", "r1"], "A group": ["r1", "r3"], "Empty": ["missing"]}
        got = pushit.group_recipes_by_category(["r3", "r1", "x", "r2"], cats, warn=True)
        assert got == {"B group": ["r1", "r2"], "A group": ["r3"], pushit.UNCATEGORISED_RECIPES: ["x"]}
        assert "'x' is not in RECIPE_CATEGORIES" in capsys.readouterr().out
        assert pushit.group_recipes_by_category(["b", "a"], {}) == {pushit.RECIPES_CATEGORY: ["a", "b"]}
        assert pushit.group_recipes_by_category([], {"A": ["a"]}) == {}
    
    
    def test_recipe_categories_make_subheadings_in_the_recipes_section():
        from datetime import datetime, timezone
        _, md = pushit.build_run_summary(
            "e", "/x.json", ok_nbs=["nb", "r_flux", "r_ts", "r_new"], failed_nbs=[], not_run_nbs=[],
            prev_committed_nbs=[], run_time=datetime(2026, 1, 1, tzinfo=timezone.utc),
            recipes={"r_flux", "r_ts", "r_new"}, recipe_categories={"Fluxes": ["r_flux"], "T and S": ["r_ts"]})
        section = md[md.index(f"### {pushit.RECIPES_CATEGORY}"):md.index("- **ESM datastore")]
        subs = [line.split(" {")[0] for line in section.splitlines() if line.startswith("#### ")]
        assert subs == ["#### Fluxes", "#### T and S", f"#### {pushit.UNCATEGORISED_RECIPES}"]
        assert section.index("`r_flux`") < section.index("#### T and S") < section.index("`r_ts`")
        assert "`nb`" not in section
        # Jump-to counts the whole recipes section.
        assert f"[{pushit.RECIPES_CATEGORY}](#cat-esmvaltool-recipes) (3)" in md
        assert '{id="cat-recipes-fluxes"}' in md
    
    
    def test_recipe_categories_nest_the_nav_group(patch_repo_paths):
        pushit.update_mkdocs_nav(ENAME, ["SST", "a", "b"], recipes={"a", "b"}, recipe_categories={"Group B": ["b"]})
        nav = yaml.safe_load((patch_repo_paths / "documentation" / "mkdocs.yml").read_text())["nav"]
        group = next(i for i in nav if pushit.RECIPES_CATEGORY in i)[pushit.RECIPES_CATEGORY]
        page = lambda nb: {nb: [{"Summary": f"pages/experiments/{ENAME}/{nb}.md"},
                                {"Recipe": f"pages/experiments/{ENAME}/recipes/{nb}.md"}]}
        assert group == [{"Group B": [page("b")]}, {pushit.UNCATEGORISED_RECIPES: [page("a")]}]
    
    
    def test_timeout_from_env_reads_minutes():
        assert recipes.timeout_from_env({}) is None
        assert recipes.timeout_from_env({"MKFIGS_RECIPE_TIMEOUT": ""}) is None
        assert recipes.timeout_from_env({"MKFIGS_RECIPE_TIMEOUT": "0"}) is None
        assert recipes.timeout_from_env({"MKFIGS_RECIPE_TIMEOUT": "nope"}) is None
        assert recipes.timeout_from_env({"MKFIGS_RECIPE_TIMEOUT": "1.5"}) == 90
    
    
    def test_run_killable_kills_the_whole_process_group(tmp_path):
        import time
        pidfile = tmp_path / "child.pid"
        start = time.monotonic()
        # The grandchild stands in for an esmvaltool worker left behind by a hung run.
        done = recipes.run_killable(["bash", "-c", f"sleep 60 & echo $! > {pidfile}; wait"], timeout=1)
        assert time.monotonic() - start < 30
        assert done.timed_out and done.returncode != 0
        child = int(pidfile.read_text())
        for _ in range(50):
            if not Path(f"/proc/{child}").exists() or "Z" in Path(f"/proc/{child}/stat").read_text().split()[2]:
                break
            time.sleep(0.1)
        else:
            raise AssertionError("grandchild still running")
        assert not recipes.run_killable(["true"], timeout=10).timed_out
    
    
    def test_timed_out_recipe_is_reported_as_failed(recipe_repo):
        repo, ofol = recipe_repo
        seen = {}
    
        def hung_runner(cmd, cwd=None, env=None, timeout=None):
            seen["timeout"] = timeout
            make_run_dir(Path(cmd[cmd.index("--output_dir") + 1]))
            done = SimpleNamespace(returncode=-9, timed_out=True)
            return done
    
        assert not recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                      timeout=120, runner=hung_runner)
        assert seen["timeout"] == 120
        marker = recipes.read_marker(ofol, "demo")
        assert "killed after 2 min" in marker["error"] and "MKFIGS_RECIPE_TIMEOUT" in marker["error"]
    
    
    def test_pinned_copy_from_an_earlier_run_is_not_reused(recipe_repo):
        repo, ofol = recipe_repo
        calls = []
        stale = ofol / "recipes" / "demo.moppy-pinned.yml"
        stale.parent.mkdir(parents=True, exist_ok=True)
        stale.write_text("old: true\n")
        assert recipes.run_recipe("demo", ofol, repo / "notebooks" / "recipes", experiment=ENAME, repo_root=repo,
                                  input_root="/g/data/ol01/run", runner=fake_runner(calls=calls))
        assert not stale.exists()
        assert calls[-1][2].endswith("demo.yml")
    `````

??? abstract "access-om3-paper-1/notebooks/mkfigs.sh (diff, +70 −3)"

    Recipe list, MOPPY_* settings, per-experiment MOPPy cache, ESMValCore data preset, recipe time limit, storage flags.

    ```diff
    --- a/notebooks/mkfigs.sh
    +++ b/notebooks/mkfigs.sh
    @@ -1,5 +1,5 @@
     #!/bin/bash
    -#PBS -l storage=scratch/x77+scratch/tm70+gdata/tm70+gdata/ik11+gdata/ol01+gdata/xp65+gdata/av17+gdata/x77+gdata/g40+gdata/v45+gdata/cj50+gdata/vk83+gdata/zv30+gdata/p73+gdata/rt52+gdata/ia39
    +#PBS -l storage=scratch/x77+scratch/tm70+gdata/tm70+gdata/ik11+gdata/ol01+gdata/xp65+gdata/av17+gdata/x77+gdata/g40+gdata/v45+gdata/cj50+gdata/vk83+gdata/zv30+gdata/p73+gdata/rt52+gdata/ia39+gdata/oi10+gdata/fs38+gdata/ct11
     #PBS -m ae
     #PBS -q normal
     #PBS -W umask=0022
    @@ -61,10 +61,21 @@
     #1. Log in to Figshare and publish the article
     #1. python3 -m mkfigs.pushit --check-figshare-upload
     
    +#
    +## ESMValTool recipes (optional) -- run alongside the notebooks, shown on the same website
    +# Recipes live in notebooks/recipes/<name>.yml (see notebooks/recipes/README.md) and are listed in the
    +# `recipes` list below. They usually need a different conda environment from the notebooks, so run them as a
    +# separate job that writes into the same mkfigs_output_<ENAME>/ folder, then run pushit once for both:
    +#1. Edit this file: the recipe list and, if needed, the MOPPY_* settings (raw-file patterns, supergrid)
    +#1. qsub mkfigs.sh                                                       # notebooks (unchanged)
    +#1. qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh   # recipes
    +#1. python3 -m mkfigs.pushit                                             # uploads / publishes both kinds
    +
     set -x
     module purge
     module use /g/data/xp65/public/modules
    -module load conda/analysis3-26.08
    +# MKFIGS_CONDA_MODULE and MKFIGS_KIND are set from the command line, with `qsub -v`, for recipe jobs.
    +module load "${MKFIGS_CONDA_MODULE:-conda/analysis3-26.08}"
     module list
     
     # ---------------------------------------------------------------------------
    @@ -199,12 +210,68 @@ array=(
     printf -v MKFIGS_NOTEBOOKS '%s:' "${array[@]}"
     MKFIGS_NOTEBOOKS="${MKFIGS_NOTEBOOKS%:}"
     
    +# ---------------------------------------------------------------------------
    +# ESMValTool recipe list (optional)
    +# Recipes are notebooks/recipes/<name>.yml. Only run when this job is submitted with MKFIGS_KIND=recipes
    +# (or "all"); a plain `qsub mkfigs.sh` ignores them. pushit lists them next to the notebooks.
    +# ---------------------------------------------------------------------------
    +
    +recipes=(
    +    om3_mlotst_basic      # minimal pipeline test: mlotst climatology map + global annual-mean timeseries
    +    om3_surface_fluxes    # pipeline test: 4 2D variables (u/v points), areacello, ESMValTool ocean scripts
    +    om3_temp_salt_vs_obs  # pipeline test: 3D thetao/so vs WOA18, EN4 and ACCESS-OM2-025 (needs data-hpc-nci)
    +    om3_arctic_ocean      # pipeline test: ESMValTool's recipe_arctic_ocean (native grid, PHC3)
    +    om3_russell18jgr_subset  # pipeline test: Russell et al. 2018 Figs 2, 3b, 3b-2 (NCL)
    +#    russell18jgr
    +)
    +
    +# ACCESS-MOPPy CMORises the raw model output the recipes read, on the fly (via python -m mkfigs.moppy_prepare,
    +# which adds the ACCESS-OM3 workarounds MOPPy doesn't have yet). Set MOPPY_INPUT_ROOT to "" to skip CMORisation
    +# (e.g. the data is already CMORised and found by ESMValTool). Default: the experiment folder holding ESMDIR.
    +# Every MOPPY_* setting can be overridden at submission, e.g. qsub -v MKFIGS_MOPPY_CACHE=/g/data/xx/cache,...
    +MOPPY_INPUT_ROOT="${MKFIGS_MOPPY_INPUT_ROOT-$(dirname "${ESMDIR}")}"
    +MOPPY_MODEL_ID="${MKFIGS_MOPPY_MODEL_ID:-ACCESS-OM3}"
    +# The run's own MOM6 supergrid (see the experiment's config): MOPPy's bundled "25 km" grid is ACCESS-OM2's.
    +MOPPY_SUPERGRID="${MKFIGS_MOPPY_SUPERGRID:-/g/data/vk83/configurations/inputs/access-om3/share/grids/global.25km/2026.06.11/ocean_hgrid.nc}"
    +MOPPY_NOMINAL_RES="${MKFIGS_MOPPY_NOMINAL_RES:-25 km}"
    +MOPPY_WORKERS="${MKFIGS_MOPPY_WORKERS:-4}"
    +# Where MOPPy finds the raw files for each CMOR variable the recipes use (COMPOUND_NAME:GLOB, relative to
    +# MOPPY_INPUT_ROOT). MOPPy's defaults follow the ACCESS-ESM1.6 archive; ACCESS-OM3 writes flat outputNNN/ files.
    +# One entry per line. 3D fields are the z-level (+z) output; Ofx.areacello comes from the static file.
    +MOPPY_PATTERNS="${MKFIGS_MOPPY_PATTERNS:-
    +Omon.mlotst:output[0-9]*/access-om3.mom6.2d.mlotst.1mon.mean.*.nc
    +Omon.hfds:output[0-9]*/access-om3.mom6.2d.hfds.1mon.mean.*.nc
    +Omon.wfo:output[0-9]*/access-om3.mom6.2d.wfo.1mon.mean.*.nc
    +Omon.tauuo:output[0-9]*/access-om3.mom6.2d.tauuo.1mon.mean.*.nc
    +Omon.tauvo:output[0-9]*/access-om3.mom6.2d.tauvo.1mon.mean.*.nc
    +Omon.thetao:output[0-9]*/access-om3.mom6.3d.thetao+z.1mon.mean.*.nc
    +Omon.so:output[0-9]*/access-om3.mom6.3d.so+z.1mon.mean.*.nc
    +Ofx.areacello:output000/access-om3.mom6.static.nc
    +}"
    +# The CMORised cache can be large: keep it on /g/data, not in $HOME. One folder per experiment: MOPPy writes
    +# every ACCESS-OM3 run to the same CMIP6 path (ACCESS-OM3/omip2/r1i1p1f1/...), so a shared cache would mix runs.
    +MOPPY_CACHE="${MKFIGS_MOPPY_CACHE:-/g/data/tm70/${USER}/moppy-esmval-cache/${ENAME}}"
    +# ESMValCore's ready-made data-source config for Gadi (CMIP replicas in oi10/fs38, observations in ct11), for
    +# recipes that compare with other models or observations. Needs those projects in the #PBS -l storage line.
    +ESMVALTOOL_PRESETS="${MKFIGS_ESMVALTOOL_PRESETS-data-hpc-nci}"
    +# Minutes allowed per `esmvaltool run` (0 = no limit). ESMValTool can hang instead of exiting when a diagnostic
    +# script fails; the run is then killed, reported FAILED and the next recipe starts.
    +RECIPE_TIMEOUT="${MKFIGS_RECIPE_TIMEOUT-120}"
    +
    +printf -v MKFIGS_RECIPES '%s:' "${recipes[@]}"
    +MKFIGS_RECIPES="${MKFIGS_RECIPES%:}"
    +
     # ---------------------------------------------------------------------------
     # Hand off to mkfigs-run (inherits the conda environment loaded above)
     # ---------------------------------------------------------------------------
    -export MKFIGS_NOTEBOOKS
    +export MKFIGS_NOTEBOOKS MKFIGS_RECIPES
    +export MKFIGS_MOPPY_INPUT_ROOT="${MOPPY_INPUT_ROOT}" MKFIGS_MOPPY_CACHE="${MOPPY_CACHE}"
    +export MKFIGS_MOPPY_MODEL_ID="${MOPPY_MODEL_ID}" MKFIGS_MOPPY_PATTERNS="${MOPPY_PATTERNS}"
    +export MKFIGS_MOPPY_SUPERGRID="${MOPPY_SUPERGRID}" MKFIGS_MOPPY_NOMINAL_RES="${MOPPY_NOMINAL_RES}"
    +export MKFIGS_MOPPY_WORKERS="${MOPPY_WORKERS}" MKFIGS_ESMVALTOOL_PRESETS="${ESMVALTOOL_PRESETS}" MKFIGS_RECIPE_TIMEOUT="${RECIPE_TIMEOUT}"
     
     exec python3 -m mkfigs.run \
    +    --kind "${MKFIGS_KIND:-notebooks}" \
         --ename "${ENAME}" \
         --esmdir "${ESMDIR}" \
         --wfolder "${WFOLDER}" \
    ```

??? abstract "access-om3-paper-1/notebooks/mkfigs_issues.py (diff, +11 −0)"

    RECIPE_CATEGORIES: sub-groups for the recipes section.

    ```diff
    --- a/notebooks/mkfigs_issues.py
    +++ b/notebooks/mkfigs_issues.py
    @@ -17,6 +17,11 @@ puts it there (this overrides issue matches); otherwise it goes to the first
     category matching one of its issue numbers; anything unmatched goes last,
     under "Miscellaneous". The left-hand nav stays a flat, alphabetical list of
     notebooks so it looks the same from run to run.
    +
    +RECIPE_CATEGORIES splits the "ESMValTool recipes" section (and nav group) into
    +sub-groups. Each entry lists recipe stems (filename in recipes/ without .yml).
    +A recipe goes under the first sub-group listing it; recipes not listed go last,
    +under "Other recipes". Leave it empty for one flat recipe table.
     """
     
     _GH = "https://github.com/ACCESS-Community-Hub/access-om3-paper-1/issues"
    @@ -81,3 +86,9 @@ CATEGORIES: dict[str, list[int | str]] = {
         "Other possible bonus figures":                  [120],
         "Miscellaneous":                                 ["GM-Testing-in-ACCESS-OM3", 154],
     }
    +
    +RECIPE_CATEGORIES: dict[str, list[str]] = {
    +    "Surface fluxes and mixed layer":      ["om3_mlotst_basic", "om3_surface_fluxes"],
    +    "Temperature and salinity structure":  ["om3_temp_salt_vs_obs"],
    +    "Regional analysis":                   ["om3_arctic_ocean", "om3_russell18jgr_subset"],
    +}
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/om3_mlotst_basic.yml (new file, +63 −0)"

    New. Level 1: mixed layer depth, model only.

    ```yaml
    # Minimal ESMValTool recipe for mkfigs: ACCESS-OM3 mixed layer depth (mlotst), CMORised on the fly by ACCESS-MOPPy.
    # Purpose: exercise the mkfigs recipe pipeline end to end (MOPPy -> ESMValTool -> mkfigs pages), not science.
    ---
    documentation:
      title: ACCESS-OM3 mixed layer depth (mlotst) - basic
      description: >
        Climatological-mean map and global-mean annual timeseries of mixed layer depth (mlotst)
        from ACCESS-OM3, CMORised from raw MOM6 output by ACCESS-MOPPy.
      authors:
        - unmaintained
      references:
        - acknow_project
    
    datasets:
      - dataset: ACCESS-OM3
        project: CMIP6
        activity: OMIP
        institute: CSIRO-COSIMA
        exp: omip2
        ensemble: r1i1p1f1
        grid: gn
        timerange: '2000/2004'
    
    preprocessors:
      climatology_map:
        climate_statistics:
          operator: mean
        regrid:
          target_grid: 1x1
          scheme: nearest
      global_annual_mean:
        regrid:
          target_grid: 1x1
          scheme: nearest
        area_statistics:
          operator: mean
        annual_statistics:
          operator: mean
    
    diagnostics:
      mlotst_map:
        description: Climatological-mean mixed layer depth.
        variables:
          mlotst:
            mip: Omon
            preprocessor: climatology_map
        scripts:
          plot:
            script: examples/diagnostic.py
            quickplot:
              plot_type: pcolormesh
              cmap: viridis
      mlotst_timeseries:
        description: Global-mean (1x1 grid) annual-mean mixed layer depth.
        variables:
          mlotst:
            mip: Omon
            preprocessor: global_annual_mean
        scripts:
          plot:
            script: examples/diagnostic.py
            quickplot:
              plot_type: plot
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/om3_surface_fluxes.yml (new file, +86 −0)"

    New. Level 2: four 2D fields incl. staggered tau, native-grid area means with areacello; vs ACCESS-ESM1-5, ACCESS-OM2(-025).

    ```yaml
    # Test recipe (level 2 of 5) for the mkfigs ESMValTool pipeline: several 2D variables CMORised in one go, two of them
    # on staggered (u/v) points, an explicit grid-cell-area (Ofx areacello) variable, area means on the native tripolar
    # grid, and ESMValTool's own ocean diagnostic scripts instead of the generic example script. Compared with
    # ACCESS-ESM1-5 (all variables) and, in the maps only, ACCESS-OM2 / ACCESS-OM2-025 (they have no hfds; their OMIP
    # model years 0348-0350 can't share a time axis with calendar years, so they stay out of the timeseries).
    # Comparison models (CMIP6 on Gadi, fs38/oi10 via the data-hpc-nci preset): ACCESS-OM2 (1 deg) and ACCESS-OM2-025
    # omip2 model years 0348-0350 = 2000-2002 (last of six 61-year JRA55-do 1958-2018 cycles), and ACCESS-ESM1-5
    # historical r1i1p1f1 (coupled, so not the same weather). No CMORised ACCESS-ESM1-6 data is on Gadi yet.
    ---
    documentation:
      title: ACCESS-OM3 surface fluxes (pipeline test)
      description: >
        Climatological maps and global-mean timeseries of net surface heat flux (hfds), net freshwater flux (wfo) and
        surface wind stress (tauuo, tauvo) from ACCESS-OM3, CMORised from raw MOM6 output by ACCESS-MOPPy, with
        ACCESS-ESM1-5 (historical) and, in the maps, ACCESS-OM2 and ACCESS-OM2-025 (omip2) for comparison.
      authors:
        - unmaintained
      references:
        - acknow_project
      # Must be present (may be empty): ESMValTool's ocean/diagnostic_*.py scripts read documentation["projects"].
      projects: []
    
    datasets:
      - dataset: ACCESS-OM3
        project: CMIP6
        activity: OMIP
        institute: CSIRO-COSIMA
        exp: omip2
        ensemble: r1i1p1f1
        grid: gn
        timerange: '2000/2002'
      - {dataset: ACCESS-ESM1-5, project: CMIP6, exp: historical, ensemble: r1i1p1f1, grid: gn, timerange: '2000/2002'}
    
    preprocessors:
      clim_map:
        climate_statistics:
          operator: mean
        regrid:
          target_grid: 1x1
          scheme: linear
      global_mean_native:
        # area mean on the native grid: needs areacello (supplementary variable) since the grid is curvilinear
        area_statistics:
          operator: mean
        annual_statistics:
          operator: mean
    
    diagnostics:
      flux_maps:
        description: Climatological-mean surface fluxes and wind stress, regridded to 1x1.
        variables:
          hfds: {mip: Omon, preprocessor: clim_map}
          wfo: &omip_map
            mip: Omon
            preprocessor: clim_map
            additional_datasets:
              - {dataset: ACCESS-OM2, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2, ensemble: r1i1p1f1,
                 grid: gn, timerange: '0348/0350'}
              - {dataset: ACCESS-OM2-025, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2, ensemble: r1i1p1f1,
                 grid: gn, timerange: '0348/0350'}
          tauuo: *omip_map
          tauvo: *omip_map
        scripts:
          maps:
            script: ocean/diagnostic_maps.py
      flux_timeseries:
        description: Global-mean annual-mean surface fluxes, area-weighted with the model's own cell areas.
        variables:
          hfds: &ts
            mip: Omon
            preprocessor: global_mean_native
            supplementary_variables:
              - short_name: areacello
                mip: Ofx
          wfo: *ts
        scripts:
          timeseries:
            script: ocean/diagnostic_timeseries.py
      cell_area:
        description: >
          No plots. ACCESS-MOPPy only reads a recipe's `variables:` (not `supplementary_variables:`), so areacello is
          listed here to have it CMORised for the area means above.
        variables:
          areacello:
            mip: Ofx
        scripts: null
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/om3_temp_salt_vs_obs.yml (new file, +93 −0)"

    New. Level 3: 3D thetao/so vs WOA18 and EN4, and vs ACCESS-ESM1-5, ACCESS-OM2(-025).

    ```yaml
    # Test recipe (level 3 of 5) for the mkfigs ESMValTool pipeline: 3D variables (thetao, so on z levels), ACCESS-OM3
    # compared with other models from NCI's CMIP6 collections (ACCESS-ESM1-5 historical; ACCESS-OM2 and ACCESS-OM2-025
    # omip2 cycle 6, whose model years 0306-0366 = 1958-2018) and with observations (WOA18, EN4) read from NCI's ESMValTool
    # observation replica. Needs ESMValCore's data-hpc-nci data sources (MKFIGS_ESMVALTOOL_PRESETS) and the
    # oi10/fs38/ct11 PBS storage flags.
    ---
    documentation:
      title: ACCESS-OM3 temperature and salinity vs observations (pipeline test)
      description: >
        Near-surface (5 m) temperature and salinity of ACCESS-OM3 and ACCESS-ESM1-5 (historical), plus ACCESS-OM2 and
        ACCESS-OM2-025 (omip2) for salinity, against the World Ocean Atlas 2018 climatology, and global-mean profiles
        against WOA18 and EN4.
      authors:
        - unmaintained
      references:
        - acknow_project
      # Must be present (may be empty): ESMValTool's ocean/diagnostic_*.py scripts read documentation["projects"].
      projects: []
    
    # custom_order: take the time mean FIRST. ESMValCore's default order runs extract_levels and regrid before
    # climate_statistics, i.e. on every monthly 3D field (22 min for one variable of ACCESS-OM2-025 on 3 years).
    preprocessors:
      surface_clim:
        custom_order: true
        climate_statistics:
          operator: mean
        extract_levels:
          levels: [5.]
          scheme: linear_extrapolate
        regrid:
          target_grid: 1x1
          scheme: linear
      global_profile:
        custom_order: true
        # annual (not climate) means: ocean/diagnostic_profiles.py groups by year, so it needs a time dimension
        annual_statistics:
          operator: mean
        regrid:
          # regular grid so area_statistics needs no areacello (not CMORised for ACCESS-OM3 in this recipe)
          target_grid: 1x1
          scheme: linear
        area_statistics:
          operator: mean
        extract_levels:
          levels: {dataset: WOA}
          scheme: linear
    
    diagnostics:
      # ACCESS-OM2 and ACCESS-OM2-025 are compared for salinity only: their published OMIP thetao files (fs38, versions
      # v20210616/v20210617) hold Kelvin values labelled "degC", ESMValCore has no fix for it, and these ocean diagnostics
      # don't guess units from the values.
      surface_vs_woa:
        description: Model, WOA18 and model minus WOA18 at 5 m (climatological mean).
        variables:
          thetao:
            mip: Omon
            preprocessor: surface_clim
            additional_datasets:
              - &woa {dataset: WOA, project: OBS6, tier: 2, type: clim, version: '2018', timerange: '2000/2000'}
          so:
            mip: Omon
            preprocessor: surface_clim
            additional_datasets:
              - *woa
              - &om2_025 {dataset: ACCESS-OM2-025, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2,
                          ensemble: r1i1p1f1, grid: gn, timerange: '0348/0350'}
              - &om2 {dataset: ACCESS-OM2, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2,
                      ensemble: r1i1p1f1, grid: gn, timerange: '0348/0350'}
        additional_datasets: &models
          - {dataset: ACCESS-OM3, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2,
             ensemble: r1i1p1f1, grid: gn, timerange: '2000/2002'}
          - {dataset: ACCESS-ESM1-5, project: CMIP6, exp: historical, ensemble: r1i1p1f1, grid: gn, timerange: '2000/2002'}
        scripts:
          model_vs_obs:
            script: ocean/diagnostic_model_vs_obs.py
            observational_dataset: {dataset: WOA, project: OBS6}
      global_profiles:
        description: Global-mean temperature and salinity profiles (climatological mean), models vs EN4 and WOA18.
        variables:
          thetao:
            mip: Omon
            preprocessor: global_profile
            additional_datasets:
              - *woa
              - &en4 {dataset: EN4, project: OBS6, tier: 2, type: reanaly, version: 4.2.2, timerange: '2000/2002'}
          so:
            mip: Omon
            preprocessor: global_profile
            additional_datasets: [*woa, *en4, *om2_025, *om2]
        additional_datasets: *models
        scripts:
          profiles:
            script: ocean/diagnostic_profiles.py
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/om3_arctic_ocean.yml (new file, +147 −0)"

    New. Level 4: ESMValTool's recipe_arctic_ocean, datasets swapped (vs PHC3 and the three ACCESS models).

    ```yaml
    # Test recipe (level 4 of 5) for the mkfigs ESMValTool pipeline: ESMValTool's own recipe_arctic_ocean.yml (2.15.0),
    # unchanged apart from the datasets (CMIP5 models -> ACCESS-OM3, ACCESS-OM2, ACCESS-OM2-025, ACCESS-ESM1-5;
    # period 1980-2005 -> 2000-2002).
    # Its diagnostic works on each model's native grid (no preprocessor), with the model's own cell areas (areacello),
    # and compares with the PHC3 climatology.
    ---
    documentation:
      title: Arctic Ocean Diagnostics
      description: |
        Arctic Ocean diagnostics.
    
      authors:
        - koldunov_nikolay
    
      maintainer:
        - koldunov_nikolay
    
      references:
        - contact_authors
    
      projects:
        - trr181
        - applicate
    
    # Model problems
    # bcc-csm1-1 - lon and lat in rotated grid coordinates, not usable
    # GFDL-ESM2G - lon and lat in rotated grid coordinates, not usable
    datasets:
      - {dataset: ACCESS-OM3, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2, ensemble: r1i1p1f1,
         grid: gn, timerange: '2000/2002'}
      - {dataset: ACCESS-OM2-025, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2, ensemble: r1i1p1f1,
         grid: gn, timerange: '0348/0350'}
      - {dataset: ACCESS-OM2, project: CMIP6, activity: OMIP, institute: CSIRO-COSIMA, exp: omip2, ensemble: r1i1p1f1,
         grid: gn, timerange: '0348/0350'}
      - {dataset: ACCESS-ESM1-5, project: CMIP6, exp: historical, ensemble: r1i1p1f1, grid: gn, timerange: '2000/2002'}
    
    diagnostics:
    
      arctic_ocean:
        description: Derive ocean variables
        variables:
          areacello:
            mip: Ofx
            additional_datasets:
              - {dataset: PHC, project: OBS6, mip: fx, tier: 2, type: clim, version: 3}   # file is OBS6_PHC_clim_3_fx_areacello.nc
          thetao: &variable
            mip: Omon
            additional_datasets:
              - {dataset: PHC, project: OBS6, mip: Omon, tier: 2, type: clim, version: 3, timerange: '1950/1950'}
          so: *variable
        scripts:
          arctic_ocean_main:
            script: arctic_ocean/arctic_ocean.py
            # Define if we calculate data for Hovmoeller diagrams.
            # This data are also needed for mean vertical profiles.
            hofm_data: True
            # Define regions, as a list.
            # 'EB' - Eurasian Basin of the Arctic Ocean
            # 'AB' - Amerasian Basin of the Arctic Ocean
            # 'Barents_sea' - Barrents Sea
            # 'North_sea'   - North Sea
            hofm_regions: ["AB" ,  'EB']
            # Define variables to use, should also be in "variables" entry of your diagnostic
            hofm_vars: ['thetao', 'so']
            # Maximum depth of Hovmoeller and vertical profiles
            hofm_depth: 1500
            # Define if Hovmoeller diagrams will be ploted.
            hofm_plot: True
            # Define colormap (as a list, same size as list with variables)
            # Only cmaps from matplotlib and cmocean are supported.
            # Additional cmap - 'custom_salinity1'.
            hofm_cmap: ['Spectral_r', 'custom_salinity1']
            # Data limits for plots,
            # List of the same size as the list of the variables
            # each entry is [vmin, vmax, number of levels, rounding limit]
            hofm_limits: [[-2, 2.3, 41, 1], [30.5, 35.1, 47, 2]]
            # Number of columns in the plot
            hofm_ncol: 3
            # Calculate the timemmean for every model
            mean: True
            # Plot vertical profiles
            profiles: True
            # Plot spatial distribution maps
            plot2d: True
            # Depths for spatial distribution maps
            plot2d_depths: [10, 100]
            # Variables to plot spatial distribution maps
            plot2d_vars: ['thetao', 'so']
            # Define colormap (as a list, same size as list with variables)
            # Only cmaps from matplotlib and cmocean are supported.
            # Additional cmap - 'custom_salinity1'.
            plot2d_cmap: ['Spectral_r', 'custom_salinity1']
            # Data limits for plots,
            # List of the same size as the list of the variables
            # each entry is [vmin, vmax, number of levels, rounding limit]
            plot2d_limits: [[-2, 4, 20, 1], [30.5, 35.1, 47, 2]]
            # number of columns for plots
            plot2d_ncol: 4
            # Plot spatial distribution of the bias plots
            plot2d_bias: True
            # Depths for spatial distribution of the bias
            plot2d_bias_depths: [10, 100]
            # Variables to plot spatial distribution of the bias for
            plot2d_bias_vars: ['thetao', 'so']
            # Color map bames for every variable
            plot2d_bias_cmap: ['balance', 'balance']
            # Data limits for plots,
            # List of the same size as the list of the variables
            # each entry is [vmin, vmax, number of levels, rounding limit]
            plot2d_bias_limits: [[-3, 3, 20, 1], [-2, 2, 47, 2]]
            # number of columns in the bias plots
            plot2d_bias_ncol: 4
            # Plot transects
            transects: true
            # Select regions (transects) to plot
            # Available options are:
            # AWpath - transect along the path of the Atlantic Water
            # Fram - Fram strait
            transects_regions: ["AWpath", "Fram"]
            # Variables to plot on transects
            transects_vars: ['thetao', 'so']
            # Color maps for every variable
            transects_cmap: ['Spectral_r', 'custom_salinity1']
            # Data limits for plots,
            # List of the same size as the list of the variables
            # each entry is [vmin, vmax, number of levels, rounding limit]
            transects_limits: [[-2, 4, 20, 1], [30.5, 35.1, 47, 2]]
            # Maximum depth to plot the data
            transects_depth: 1500
            # number of columns
            transects_ncol: 3
            # Calculate Atlantic Water core depth and temperature
            # one has to have  `hofm_data`: True and "EB" as one of the `hofm_regions`
            AW_core: True
            # Plot maps of temperature spatial distribution at the depth
            # of Atlantic Water depth
            AW_core_2d: True
            # Plot TS diagrams
            tsdiag: True
            # Regions to plot TS diagrams for
            tsdiag_regions: ["AB" ,  'EB']
            # Maximum depth to consider data for TS diagrams
            tsdiag_depth: 1500
            # Number of columns
            tsdiag_ncol: 3
            # Observational dataset
            observational_dataset: {dataset: PHC, project: OBS,}
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/om3_russell18jgr_subset.yml (new file, +150 −0)"

    New. Level 5: three NCL figures of recipe_russell18jgr (Southern Ocean wind stress and fronts).

    ```yaml
    # Test recipe (level 5 of 5) for the mkfigs ESMValTool pipeline: three diagnostics of ESMValTool's
    # recipe_russell18jgr.yml (2.15.0), which are NCL scripts working on each model's native grid. Diagnostics and script
    # settings are upstream's; only the datasets changed (CMIP5 historical 1986-2005 -> ACCESS-OM3 + ACCESS-OM2-025 OMIP2,
    # 2000-2002). Figures needing variables ACCESS-MOPPy can't make from ACCESS-OM3 yet (sic, fgco2, ph, volcello) are left out.
    ---
    documentation:
      title: Southern Ocean metrics (Russell et al. 2018), Figures 2, 3b and 3b-2
      description: 'Subset of ESMValTool''s recipe_russell18jgr.yml (NCL diagnostics): zonal-mean zonal wind stress
        and the positions of the Subantarctic and Polar Fronts, for ACCESS-OM3 and ACCESS-OM2-025.'
      authors:
      - russell_joellen
      - pandde_amarjiit
      maintainer:
      - russell_joellen
      - pandde_amarjiit
      references:
      - russell18jgr
      projects:
      - russell_project
    preprocessors:
      preprocessor_time_land:
        climate_statistics:
          operator: mean
          period: full
        mask_landsea:
          mask_out: land
      preprocessor_time:
        climate_statistics:
          operator: mean
          period: full
      # Figures 3b/3b-2 have no preprocessor upstream, so NCL loads every monthly 3D field (>30 GB for 3 years of 1-degree
      # ACCESS-OM2; far more at 25 km). The scripts average over their first (time) dimension themselves, so keep a time
      # dimension (annual means, not climate_statistics) and only the upper 500 m (fronts use the 400 m and 0-200 m levels).
      preprocessor_fronts:
        extract_volume:
          z_min: 0
          z_max: 500
        annual_statistics:
          operator: mean
    diagnostics:
      Figure_2:
        description: 'Diagnostic for russell et al Figure 2. Plots The zonal and annual means
    
          of the zonal wind stress (N/m^2). Here, we are using tauuo,
    
          figures in russell18jgr paper were  made using tauuo, but tauu can
    
          also be used if tauuo file is not available. To use tauu variable
    
          in this recipe just uncomment variable ''tauu'' and add dataset names
    
          in additional datasets of respective variable. If there are no dataset
    
          in a variable, entire variable section needs to be commented out.
    
          '
        themes:
        - phys
        realms:
        - ocean
        variables:
          tauuo:
            preprocessor: preprocessor_time
            mip: Omon
        scripts:
          tauu-lineplot:
            script: russell18jgr/russell18jgr-fig2.ncl
            styleset: CMIP5
            ncdf: default
        additional_datasets: &id001
        - dataset: ACCESS-OM3
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
        - dataset: ACCESS-OM2-025
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
        - dataset: ACCESS-OM2
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
        - dataset: ACCESS-ESM1-5
          project: CMIP6
          exp: historical
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
      Figure_3b:
        description: 'Diagnostic for russell et al figure 3b. Plots the latitudinal position of
    
          Subantarctic Front. Using definitions from Orsi et al (1995), the
    
          Subantarctic Front is defined here as the poleward location of
    
          the 4C(277.15K) isotherm at(closest to and less than) 400 m.
    
          '
        themes:
        - phys
        realms:
        - ocean
        variables:
          thetao:
            mip: Omon
            preprocessor: preprocessor_fronts
        scripts:
          Subantarctic-Front:
            script: russell18jgr/russell18jgr-fig3b.ncl
            styleset: CMIP5
            ncdf: CMIP5
        additional_datasets: *id001
      Figure_3b-2:
        description: 'Diagnostic for russell et al figure 3b-2 (Polar fronts). Plots the
    
          latitudinal position of Polar Front. Using definitions from Orsi et
    
          al. (1995). The Polar Front is defined here as the poleward location
    
          of the 2C (275.15K) isotherm of the temperature minimum between 0 and
    
          200 m (closest to and less than 200m).
    
          '
        themes:
        - phys
        realms:
        - ocean
        variables:
          thetao:
            mip: Omon
            preprocessor: preprocessor_fronts
        scripts:
          Polar-front:
            script: russell18jgr/russell18jgr-fig3b-2.ncl
            styleset: CMIP5
            ncdf: default
        additional_datasets: *id001
    ```

??? abstract "access-om3-paper-1/notebooks/recipes/README.md (new file, +62 −0)"

    New. How to add and run a recipe; the workaround table.

    ````markdown
    # ESMValTool recipes
    
    Recipes in this folder are run by `mkfigs.sh` next to the evaluation notebooks and shown on the same
    website: the landing-page table lists a recipe like a notebook, its **Figures** page holds the plots
    ESMValTool produced, and its **Recipe** page shows the recipe that ran (ESMValTool's resolved
    `recipe_filled.yml`) with the tool versions.
    
    *Status: draft. `om3_mlotst_basic` has run end to end on Gadi (ACCESS-OM3 25 km output, conda/analysis3-26.11:
    ACCESS-MOPPy 1.9.0, ESMValTool 2.15.0). It is a pipeline test, not an evaluation recipe.*
    
    ## Adding a recipe
    
    1. Put the recipe in this folder as `<name>.yml`. The file name (without `.yml`) is the stem used everywhere:
       the plot files (`<name>_01.png`, ...), the pages, the nav entry and the issue/category tables.
       Use a stem that is not a prefix of another notebook or recipe name, so figures are never attributed to the wrong one.
    2. Add `<name>` to the `recipes` list in `../mkfigs.sh`.
    3. Optionally add the stem to `ISSUES` / `CATEGORIES` in `../mkfigs_issues.py`, exactly as for a notebook.
    4. Run it as a separate job (recipes usually need a different conda environment from the notebooks):
    
       ```bash
       cd notebooks
       qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh
       python3 -m mkfigs.pushit          # on a login node, once the job has finished
       ```
    
       Output goes to `mkfigs_output_<ENAME>/`, the same folder the notebooks use: plots as
       `mkmd/<name>_NN.png`, `mkmd/<name>.md`, `<name>_recipe_run.json` (the run marker) and
       `<name>_recipe_filled.yml`. Failures are listed in `mkmd/mkfigs_recipe_errors.log`.
    
    ## What the runner does
    
    For each recipe it copies the file into `mkfigs_output_<ENAME>/recipes/`, CMORises the model output it needs
    with ACCESS-MOPPy (only if `MOPPY_INPUT_ROOT` is set in `mkfigs.sh`), runs `esmvaltool run`, then collects the
    plots. Extra `esmvaltool run` arguments can be passed with the `MKFIGS_ESMVALTOOL_ARGS` environment variable,
    for example `qsub -v MKFIGS_KIND=recipes,MKFIGS_ESMVALTOOL_ARGS="--max-parallel-tasks 4" mkfigs.sh`.
    Each `esmvaltool run` is limited to `RECIPE_TIMEOUT` minutes (default 120, `MKFIGS_RECIPE_TIMEOUT` to change, 0 for
    no limit): ESMValTool can hang instead of exiting when a diagnostic script fails, so the run is killed, reported as
    FAILED, and the next recipe starts.
    
    CMORisation runs `python -m mkfigs.moppy_prepare`, which is MOPPy's `moppy-esmval-prepare` plus workarounds
    for things MOPPy only supports for ACCESS-ESM1.6 so far (each one should become an upstream MOPPy fix):
    
    | `mkfigs.sh` setting | Why |
    |---|---|
    | `MOPPY_MODEL_ID=ACCESS-OM3` | MOPPy only CMORises recipe datasets named ACCESS-ESM1-5/1-6, and the CMIP6 CV has no ACCESS-OM3 (mkfigs adds a provisional entry copied from ACCESS-OM2, with ACCESS-OM2's model descriptions replaced by a "provisional" note so the files' `source` attribute doesn't describe MOM5). |
    | `MOPPY_PATTERNS` | MOPPy's default raw-file globs follow the ESM1.6 archive; ACCESS-OM3 writes `outputNNN/access-om3.mom6.2d.<var>.1mon.mean.<YYYY>.nc`. One `COMPOUND_NAME:GLOB` per variable, one per line (or `;`-separated). |
    | `MOPPY_SUPERGRID`, `MOPPY_NOMINAL_RES` | MOPPy's bundled "25 km" supergrid is ACCESS-OM2's (1080 rows); ACCESS-OM3 25 km has 1152. Use the run's own `ocean_hgrid.nc`. |
    | (automatic) | MOPPy writes a `<recipe>.moppy-pinned.yml` copy that pins `version:` on every ACCESS-ESM1-5/1-6 entry, so published ACCESS-ESM1-5 data used for comparison was looked up under the CMORisation date and not found. mkfigs turns the pinning off; ESMValCore already uses the newest version of each variable. |
    | (automatic) | 3D fields: MOPPy's ACCESS-OM3 mapping knows MOM6's native vertical axis `zl`, not the `z_l`/`z_i` of the z-remapped `+z` output, so the depth axis stayed un-CMORised (and last) and ESMValCore rejected the files ("olevel: does not exist"). mkfigs reads `z_l`/`z_i` as `zl`/`zi`. The files still have no `lev_bnds` (MOPPy issue; ESMValCore accepts it). |
    | `MOPPY_CACHE` | Where the CMORised files go. Default `/g/data/tm70/$USER/moppy-esmval-cache/<ENAME>`: one folder per experiment, because MOPPy writes every ACCESS-OM3 run to the same CMIP6 path (`ACCESS-OM3/omip2/r1i1p1f1/...`). |
    | (automatic) | MOPPy asks for creator details on stdin if `~/.moppy/user.yml` is missing, which kills a batch job; mkfigs writes a generic profile to `mkfigs_output_<ENAME>/moppy_user.yml` instead (`MKFIGS_MOPPY_USER_CONFIG` to use your own). |
    
    Only variables with a direct MOPPy mapping from raw ACCESS-OM3 output work today. Directly mapped monthly
    `Omon` variables found in the test run: hfds, wfo, pbo, mlotst, tauuo, tauvo, pso, hfrainds, friver, sfdsi, thkcello.
    (`zos` is mapped from a raw `SSH` field that ACCESS-OM3 doesn't write; there is no monthly-mean `tos`.)
    
    ## Still to confirm
    
    - Institution: the CMIP6 CV has no ACCESS-NRI, so `om3_mlotst_basic.yml` uses `institute: CSIRO-COSIMA` as a placeholder.
    - Which recipe to run next. The ACCESS-OM3/CM3 Southern Ocean comparison in `ACCESS-Evaluation-recipes`
      (`web/results/russell-1986-2005`) uses `recipe_russell18jgr`, which needs the Python port in ESMValTool PR 4612
      (not in the released ESMValTool 2.15.0).
    ````

