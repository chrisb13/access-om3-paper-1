{% raw %}
# om3_mlotst_basic

<!-- recipes-review-banner -->
!!! warning "Preview: ESMValTool recipe support (not merged)"
    This site is a preview from a fork. The ESMValTool recipes and their pages come from draft pull requests to
    [access-model-mkfigs](https://github.com/ACCESS-NRI/access-model-mkfigs/pulls?q=is%3Apr+esmvaltool) and [access-om3-paper-1](https://github.com/ACCESS-Community-Hub/access-om3-paper-1/pulls?q=is%3Apr+esmvaltool).
    [Read the implementation review](../../../recipes-review.md) and comment on the pull requests.


ESMValTool recipe used to produce the figures for experiment **MC_25km_jra_iaf+wombatlite-test5.1-8f50b769**. [Back to the figures](../om3_mlotst_basic.md).

Run finished: 2026-11-11 14:23 UTC

| Tool | Version |
|---|---|
| ESMValCore | 2.15.0 |
| ESMValTool | 2.15.0 |
| access-moppy | 1.9.0 |

This is ESMValTool's resolved copy of the recipe (`<recipe>_filled.yml`), i.e. what actually ran.

```yaml
documentation:
  title: ACCESS-OM3 mixed layer depth (mlotst) - basic
  description: Climatological-mean map and global-mean annual timeseries of mixed
    layer depth (mlotst) from ACCESS-OM3, CMORised from raw MOM6 output by ACCESS-MOPPy.
  authors:
  - unmaintained
  references:
  - acknow_project
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
  default: {}
diagnostics:
  mlotst_map:
    description: Climatological-mean mixed layer depth.
    variables:
      mlotst:
        mip: Omon
        preprocessor: climatology_map
        project: CMIP6
        activity: OMIP
        institute: CSIRO-COSIMA
        exp: omip2
        ensemble: r1i1p1f1
        grid: gn
        timerange: 2000/2004
        version: v20261111
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
        project: CMIP6
        activity: OMIP
        institute: CSIRO-COSIMA
        exp: omip2
        ensemble: r1i1p1f1
        grid: gn
        timerange: 2000/2004
        version: v20261111
        supplementary_variables:
        - mip: Ofx
          short_name: areacello
          version: v20261112
    scripts:
      plot:
        script: examples/diagnostic.py
        quickplot:
          plot_type: plot
datasets:
- dataset: ACCESS-OM3
```

## Reproduce these plots

These commands repeat this run by hand: the same recipe, input data and settings, without mkfigs' page building. Run them where the tool versions above are installed; the cached CMORised data can be reused for later runs.

```bash
# On Gadi, in a clone of this repository made with --recurse-submodules:
module use /g/data/xp65/public/modules && module load conda/analysis3-26.11
export PYTHONPATH=$PWD/external/access-model-mkfigs/src
WORK=/g/data/$PROJECT/$USER/om3_mlotst_basic   # any folder you can write to
mkdir -p $WORK/esmvaltool_config
esmvaltool config copy data-hpc-nci.yml --target_file $WORK/esmvaltool_config/data-hpc-nci.yml   # ESMValCore data preset

# 1. CMORise the model output the recipe needs (ACCESS-MOPPy)
python -m mkfigs.moppy_prepare notebooks/recipes/om3_mlotst_basic.yml \
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
ESMVALTOOL_CONFIG_DIR=$WORK/esmvaltool_config esmvaltool run notebooks/recipes/om3_mlotst_basic.yml --output_dir $WORK/output --max-parallel-tasks 4

# Plots: $WORK/output/om3_mlotst_basic_<YYYYMMDD_HHMMSS>/plots/
```

To remake the whole website entry instead, set `ENAME`/`ESMDIR` in `notebooks/mkfigs.sh` and run `qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh` from `notebooks/`, then `python3 -m mkfigs.pushit`.

{% endraw %}
