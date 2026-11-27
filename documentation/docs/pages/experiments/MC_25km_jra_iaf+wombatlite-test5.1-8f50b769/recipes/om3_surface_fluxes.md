{% raw %}
# om3_surface_fluxes

<!-- recipes-review-banner -->
!!! warning "Preview: ESMValTool recipe support (not merged)"
    This site is a preview from a fork. The ESMValTool recipes and their pages come from draft pull requests to
    [access-model-mkfigs](https://github.com/ACCESS-NRI/access-model-mkfigs/pulls?q=is%3Apr+esmvaltool) and [access-om3-paper-1](https://github.com/ACCESS-Community-Hub/access-om3-paper-1/pulls?q=is%3Apr+esmvaltool).
    [Read the implementation review](../../../recipes-review.md) and comment on the pull requests.


ESMValTool recipe used to produce the figures for experiment **MC_25km_jra_iaf+wombatlite-test5.1-8f50b769**. [Back to the figures](../om3_surface_fluxes.md).

Run finished: 2026-11-11 14:24 UTC

| Tool | Version |
|---|---|
| ESMValCore | 2.15.0 |
| ESMValTool | 2.15.0 |
| access-moppy | 1.9.0 |

This is ESMValTool's resolved copy of the recipe (`<recipe>_filled.yml`), i.e. what actually ran.

```yaml
documentation:
  title: ACCESS-OM3 surface fluxes (pipeline test)
  description: Climatological maps and global-mean timeseries of net surface heat
    flux (hfds), net freshwater flux (wfo) and surface wind stress (tauuo, tauvo)
    from ACCESS-OM3, CMORised from raw MOM6 output by ACCESS-MOPPy, with ACCESS-ESM1-5
    (historical) and, in the maps, ACCESS-OM2 and ACCESS-OM2-025 (omip2) for comparison.
  authors:
  - unmaintained
  references:
  - acknow_project
  projects: []
preprocessors:
  clim_map:
    climate_statistics:
      operator: mean
    regrid:
      target_grid: 1x1
      scheme: linear
  global_mean_native:
    area_statistics:
      operator: mean
    annual_statistics:
      operator: mean
  default: {}
diagnostics:
  flux_maps:
    description: Climatological-mean surface fluxes and wind stress, regridded to
      1x1.
    variables:
      hfds:
        mip: Omon
        preprocessor: clim_map
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        timerange: 2000/2002
        additional_datasets:
        - dataset: ACCESS-OM3
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          version: v20261112
        - dataset: ACCESS-ESM1-5
          exp: historical
          version: v20191115
      wfo:
        mip: Omon
        preprocessor: clim_map
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        additional_datasets:
        - dataset: ACCESS-OM2
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210616
        - dataset: ACCESS-OM2-025
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210617
        - dataset: ACCESS-OM3
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 2000/2002
          version: v20261112
        - dataset: ACCESS-ESM1-5
          exp: historical
          timerange: 2000/2002
          version: v20191115
      tauuo:
        mip: Omon
        preprocessor: clim_map
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        additional_datasets:
        - dataset: ACCESS-OM2
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210616
        - dataset: ACCESS-OM2-025
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210617
        - dataset: ACCESS-OM3
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 2000/2002
          version: v20261112
        - dataset: ACCESS-ESM1-5
          exp: historical
          timerange: 2000/2002
          version: v20191115
      tauvo:
        mip: Omon
        preprocessor: clim_map
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        additional_datasets:
        - dataset: ACCESS-OM2
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210616
        - dataset: ACCESS-OM2-025
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 0348/0350
          version: v20210617
        - dataset: ACCESS-OM3
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          timerange: 2000/2002
          version: v20261112
        - dataset: ACCESS-ESM1-5
          exp: historical
          timerange: 2000/2002
          version: v20191115
    scripts:
      maps:
        script: ocean/diagnostic_maps.py
  flux_timeseries:
    description: Global-mean annual-mean surface fluxes, area-weighted with the model's
      own cell areas.
    variables:
      hfds:
        mip: Omon
        preprocessor: global_mean_native
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        timerange: 2000/2002
        supplementary_variables:
        - mip: Ofx
          short_name: areacello
      wfo:
        mip: Omon
        preprocessor: global_mean_native
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
        timerange: 2000/2002
        supplementary_variables:
        - mip: Ofx
          short_name: areacello
    scripts:
      timeseries:
        script: ocean/diagnostic_timeseries.py
    additional_datasets:
    - dataset: ACCESS-OM3
      activity: OMIP
      institute: CSIRO-COSIMA
      exp: omip2
      version: v20261112
    - dataset: ACCESS-ESM1-5
      exp: historical
      version: v20191115
  cell_area:
    description: 'No plots. ACCESS-MOPPy only reads a recipe''s `variables:` (not
      `supplementary_variables:`), so areacello is listed here to have it CMORised
      for the area means above.

      '
    variables:
      areacello:
        mip: Ofx
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
    scripts: null
    additional_datasets:
    - dataset: ACCESS-OM3
      activity: OMIP
      institute: CSIRO-COSIMA
      exp: omip2
      version: v20261112
    - dataset: ACCESS-ESM1-5
      exp: historical
      version: v20191115
datasets: []
```

## Reproduce these plots

These commands repeat this run by hand: the same recipe, input data and settings, without mkfigs' page building. Run them where the tool versions above are installed; the cached CMORised data can be reused for later runs.

```bash
# On Gadi, in a clone of this repository made with --recurse-submodules:
module use /g/data/xp65/public/modules && module load conda/analysis3-26.11
export PYTHONPATH=$PWD/external/access-model-mkfigs/src
WORK=/g/data/$PROJECT/$USER/om3_surface_fluxes   # any folder you can write to
mkdir -p $WORK/esmvaltool_config
esmvaltool config copy data-hpc-nci.yml --target_file $WORK/esmvaltool_config/data-hpc-nci.yml   # ESMValCore data preset

# 1. CMORise the model output the recipe needs (ACCESS-MOPPy)
python -m mkfigs.moppy_prepare notebooks/recipes/om3_surface_fluxes.yml \
    --input-root /g/data/ol01/outputs/access-om3-25km/MC_25km_jra_iaf+wombatlite-test5.1-8f50b769 \
    --cache-dir $WORK/cmorised \
    --output-config $WORK/esmvaltool_config/moppy-esmval-data.yml \
    --model-id ACCESS-OM3 \
    --pattern 'Omon.mlotst:output[0-9]*/access-om3.mom6.2d.mlotst.1mon.mean.*.nc' \
    --pattern 'Omon.hfds:output[0-9]*/access-om3.mom6.2d.hfds.1mon.mean.*.nc' \
    --pattern 'Omon.wfo:output[0-9]*/access-om3.mom6.2d.wfo.1mon.mean.*.nc' \
    --pattern 'Omon.tauuo:output[0-9]*/access-om3.mom6.2d.tauuo.1mon.mean.*.nc' \
    --pattern 'Omon.tauvo:output[0-9]*/access-om3.mom6.2d.tauvo.1mon.mean.*.nc' \
    --pattern 'Omon.thetao:output[0-9]*/access-om3.mom6.3d.thetao+z.1mon.mean.*.nc' \
    --pattern 'Omon.so:output[0-9]*/access-om3.mom6.3d.so+z.1mon.mean.*.nc' \
    --pattern Ofx.areacello:output000/access-om3.mom6.static.nc \
    --supergrid /g/data/vk83/configurations/inputs/access-om3/share/grids/global.25km/2026.06.11/ocean_hgrid.nc \
    --nominal-resolution '25 km' \
    --workers 4 \
    --user-config $WORK/moppy_user.yml

# 2. Run the recipe (use the .moppy-pinned.yml copy instead if step 1 wrote one next to it)
ESMVALTOOL_CONFIG_DIR=$WORK/esmvaltool_config esmvaltool run notebooks/recipes/om3_surface_fluxes.yml --output_dir $WORK/output --max-parallel-tasks 4

# Plots: $WORK/output/om3_surface_fluxes_<YYYYMMDD_HHMMSS>/plots/
```

To remake the whole website entry instead, set `ENAME`/`ESMDIR` in `notebooks/mkfigs.sh` and run `qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh` from `notebooks/`, then `python3 -m mkfigs.pushit`.

{% endraw %}
