{% raw %}
# om3_russell18jgr_subset

<!-- recipes-review-banner -->
!!! warning "Preview: ESMValTool recipe support (not merged)"
    This site is a preview from a fork. The ESMValTool recipes and their pages come from draft pull requests to
    [access-model-mkfigs](https://github.com/ACCESS-NRI/access-model-mkfigs/pulls?q=is%3Apr+esmvaltool) and [access-om3-paper-1](https://github.com/ACCESS-Community-Hub/access-om3-paper-1/pulls?q=is%3Apr+esmvaltool).
    [Read the implementation review](../../../recipes-review.md) and comment on the pull requests.


ESMValTool recipe used to produce the figures for experiment **MC_25km_jra_iaf+wombatlite-test5.1-8f50b769**. [Back to the figures](../om3_russell18jgr_subset.md).

Run finished: 2026-11-11 15:15 UTC

| Tool | Version |
|---|---|
| ESMValCore | 2.15.0 |
| ESMValTool | 2.15.0 |
| access-moppy | 1.9.0 |

This is ESMValTool's resolved copy of the recipe (`<recipe>_filled.yml`), i.e. what actually ran.

```yaml
documentation:
  title: Southern Ocean metrics (Russell et al. 2018), Figures 2, 3b and 3b-2
  description: 'Subset of ESMValTool''s recipe_russell18jgr.yml (NCL diagnostics):
    zonal-mean zonal wind stress and the positions of the Subantarctic and Polar Fronts,
    for ACCESS-OM3 and ACCESS-OM2-025.'
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
  preprocessor_fronts:
    extract_volume:
      z_min: 0
      z_max: 500
    annual_statistics:
      operator: mean
  default: {}
diagnostics:
  Figure_2:
    description: 'Diagnostic for russell et al Figure 2. Plots The zonal and annual
      means

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
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
    scripts:
      tauu-lineplot:
        script: russell18jgr/russell18jgr-fig2.ncl
        styleset: CMIP5
        ncdf: default
  Figure_3b:
    description: 'Diagnostic for russell et al figure 3b. Plots the latitudinal position
      of

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
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
    scripts:
      Subantarctic-Front:
        script: russell18jgr/russell18jgr-fig3b.ncl
        styleset: CMIP5
        ncdf: CMIP5
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
        project: CMIP6
        ensemble: r1i1p1f1
        grid: gn
    scripts:
      Polar-front:
        script: russell18jgr/russell18jgr-fig3b-2.ncl
        styleset: CMIP5
        ncdf: default
datasets:
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
```

## Reproduce these plots

These commands repeat this run by hand: the same recipe, input data and settings, without mkfigs' page building. Run them where the tool versions above are installed; the cached CMORised data can be reused for later runs.

```bash
# On Gadi, in a clone of this repository made with --recurse-submodules:
module use /g/data/xp65/public/modules && module load conda/analysis3-26.11
export PYTHONPATH=$PWD/external/access-model-mkfigs/src
WORK=/g/data/$PROJECT/$USER/om3_russell18jgr_subset   # any folder you can write to
mkdir -p $WORK/esmvaltool_config
esmvaltool config copy data-hpc-nci.yml --target_file $WORK/esmvaltool_config/data-hpc-nci.yml   # ESMValCore data preset

# 1. CMORise the model output the recipe needs (ACCESS-MOPPy)
python -m mkfigs.moppy_prepare notebooks/recipes/om3_russell18jgr_subset.yml \
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
ESMVALTOOL_CONFIG_DIR=$WORK/esmvaltool_config esmvaltool run notebooks/recipes/om3_russell18jgr_subset.yml --output_dir $WORK/output --max-parallel-tasks 4

# Plots: $WORK/output/om3_russell18jgr_subset_<YYYYMMDD_HHMMSS>/plots/
```

To remake the whole website entry instead, set `ENAME`/`ESMDIR` in `notebooks/mkfigs.sh` and run `qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh` from `notebooks/`, then `python3 -m mkfigs.pushit`.

{% endraw %}
