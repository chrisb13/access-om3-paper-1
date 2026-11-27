{% raw %}
# om3_arctic_ocean

<!-- recipes-review-banner -->
!!! warning "Preview: ESMValTool recipe support (not merged)"
    This site is a preview from a fork. The ESMValTool recipes and their pages come from draft pull requests to
    [access-model-mkfigs](https://github.com/ACCESS-NRI/access-model-mkfigs/pulls?q=is%3Apr+esmvaltool) and [access-om3-paper-1](https://github.com/ACCESS-Community-Hub/access-om3-paper-1/pulls?q=is%3Apr+esmvaltool).
    [Read the implementation review](../../../recipes-review.md) and comment on the pull requests.


ESMValTool recipe used to produce the figures for experiment **MC_25km_jra_iaf+wombatlite-test5.1-8f50b769**. [Back to the figures](../om3_arctic_ocean.md).

Run finished: 2026-11-11 15:12 UTC

| Tool | Version |
|---|---|
| ESMValCore | 2.15.0 |
| ESMValTool | 2.15.0 |
| access-moppy | 1.9.0 |

This is ESMValTool's resolved copy of the recipe (`<recipe>_filled.yml`), i.e. what actually ran.

```yaml
documentation:
  title: Arctic Ocean Diagnostics
  description: Arctic Ocean diagnostics.
  authors:
  - koldunov_nikolay
  maintainer:
  - koldunov_nikolay
  references:
  - contact_authors
  projects:
  - trr181
  - applicate
diagnostics:
  arctic_ocean:
    description: Derive ocean variables
    variables:
      areacello:
        additional_datasets:
        - mip: Ofx
          dataset: ACCESS-OM2
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          version: v20210616
        - mip: Ofx
          dataset: ACCESS-OM2-025
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          version: v20210617
        - mip: Ofx
          dataset: ACCESS-OM3
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          version: v20261112
        - mip: Ofx
          dataset: ACCESS-ESM1-5
          project: CMIP6
          exp: historical
          ensemble: r1i1p1f1
          grid: gn
          version: v20191115
        - mip: fx
          dataset: PHC
          project: OBS6
          tier: 2
          type: clim
          version: '3'
      thetao:
        mip: Omon
        additional_datasets:
        - dataset: ACCESS-OM2
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
          version: v20210616
        - dataset: ACCESS-OM2-025
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
          version: v20210617
        - dataset: ACCESS-OM3
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
          version: v20261112
        - dataset: ACCESS-ESM1-5
          project: CMIP6
          exp: historical
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
          version: v20191115
        - dataset: PHC
          project: OBS6
          tier: 2
          type: clim
          version: '3'
          timerange: 1950/1950
      so:
        mip: Omon
        additional_datasets:
        - dataset: ACCESS-OM2
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
          version: v20210616
        - dataset: ACCESS-OM2-025
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 0348/0350
          version: v20210617
        - dataset: ACCESS-OM3
          project: CMIP6
          activity: OMIP
          institute: CSIRO-COSIMA
          exp: omip2
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
          version: v20261112
        - dataset: ACCESS-ESM1-5
          project: CMIP6
          exp: historical
          ensemble: r1i1p1f1
          grid: gn
          timerange: 2000/2002
          version: v20191115
        - dataset: PHC
          project: OBS6
          tier: 2
          type: clim
          version: '3'
          timerange: 1950/1950
    scripts:
      arctic_ocean_main:
        script: arctic_ocean/arctic_ocean.py
        hofm_data: true
        hofm_regions:
        - AB
        - EB
        hofm_vars:
        - thetao
        - so
        hofm_depth: 1500
        hofm_plot: true
        hofm_cmap:
        - Spectral_r
        - custom_salinity1
        hofm_limits:
        - - -2
          - 2.3
          - 41
          - 1
        - - 30.5
          - 35.1
          - 47
          - 2
        hofm_ncol: 3
        mean: true
        profiles: true
        plot2d: true
        plot2d_depths:
        - 10
        - 100
        plot2d_vars:
        - thetao
        - so
        plot2d_cmap:
        - Spectral_r
        - custom_salinity1
        plot2d_limits:
        - - -2
          - 4
          - 20
          - 1
        - - 30.5
          - 35.1
          - 47
          - 2
        plot2d_ncol: 4
        plot2d_bias: true
        plot2d_bias_depths:
        - 10
        - 100
        plot2d_bias_vars:
        - thetao
        - so
        plot2d_bias_cmap:
        - balance
        - balance
        plot2d_bias_limits:
        - - -3
          - 3
          - 20
          - 1
        - - -2
          - 2
          - 47
          - 2
        plot2d_bias_ncol: 4
        transects: true
        transects_regions:
        - AWpath
        - Fram
        transects_vars:
        - thetao
        - so
        transects_cmap:
        - Spectral_r
        - custom_salinity1
        transects_limits:
        - - -2
          - 4
          - 20
          - 1
        - - 30.5
          - 35.1
          - 47
          - 2
        transects_depth: 1500
        transects_ncol: 3
        AW_core: true
        AW_core_2d: true
        tsdiag: true
        tsdiag_regions:
        - AB
        - EB
        tsdiag_depth: 1500
        tsdiag_ncol: 3
        observational_dataset:
          dataset: PHC
          project: OBS
datasets: []
```

## Reproduce these plots

These commands repeat this run by hand: the same recipe, input data and settings, without mkfigs' page building. Run them where the tool versions above are installed; the cached CMORised data can be reused for later runs.

```bash
# On Gadi, in a clone of this repository made with --recurse-submodules:
module use /g/data/xp65/public/modules && module load conda/analysis3-26.11
export PYTHONPATH=$PWD/external/access-model-mkfigs/src
WORK=/g/data/$PROJECT/$USER/om3_arctic_ocean   # any folder you can write to
mkdir -p $WORK/esmvaltool_config
esmvaltool config copy data-hpc-nci.yml --target_file $WORK/esmvaltool_config/data-hpc-nci.yml   # ESMValCore data preset

# 1. CMORise the model output the recipe needs (ACCESS-MOPPy)
python -m mkfigs.moppy_prepare notebooks/recipes/om3_arctic_ocean.yml \
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
ESMVALTOOL_CONFIG_DIR=$WORK/esmvaltool_config esmvaltool run notebooks/recipes/om3_arctic_ocean.yml --output_dir $WORK/output --max-parallel-tasks 4

# Plots: $WORK/output/om3_arctic_ocean_<YYYYMMDD_HHMMSS>/plots/
```

To remake the whole website entry instead, set `ENAME`/`ESMDIR` in `notebooks/mkfigs.sh` and run `qsub -v MKFIGS_KIND=recipes,MKFIGS_CONDA_MODULE=conda/analysis3-26.11 mkfigs.sh` from `notebooks/`, then `python3 -m mkfigs.pushit`.

{% endraw %}
