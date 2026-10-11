# Running the models

Every live model has a `run-*.sh` wrapper in the project root. Each one changes to the project
root, sources `ssl-env.sh`, then calls `uv run python -m src.models.<model>.run`, passing every
argument straight through. Call them from any directory.

**Each script's header states what it reads and in what order, and the block at the bottom lists
that model's options with a note on what each is for.** Those headers are the authority; this
file is the map.

## Run order

`expectations` first: most models read its saved output rather than re-estimating the signal.
Beyond that the chains are short.

| Model | Command | Needs first |
|-------|---------|-------------|
| Expectations | `./run-expectations.sh` | — |
| y\* | `./run-ystar.sh` | — |
| u\* | `./run-ustar.sh` | `expectations`, `ystar` |
| Joint y\*/u\* | `./run-ystar-ustar.sh` | `expectations` |
| Cobb-Douglas | `./run-cd.sh` | — |
| r\* from bonds | `./run-rstar-bonds.sh` | `expectations` |
| r\* from the RBA | `./run-rstar-rba.sh` | `expectations` |
| r\* semi-structural | `./run-rstar-qpm.sh` | `expectations` |
| GDP nowcasts | `./run-gdp-nowcast-{bridge,dfm,bvar,components}.sh` | — |
| Bank costs, participation, TWI gap | `./run-bank-costs.sh`, `./run-participation-rate.sh`, `./run-twi-gap.sh` | — |
| NAIRU, HLW, TVP-VAR *(retired)* | `./run-nairu.sh`, `./run-rstar-hlw.sh`, `./run-rstar-tvpvar.sh` | `expectations` |
| r\* by inversion *(retired)* | `./run-rstar-invert.sh` | `expectations`, `ystar_ustar` |

`ustar` loads the output gap and its posterior sd from a completed `ystar` run, and raises a
clear error if none is saved — even though its Okun equation is off by default, so no default
equation uses the gap. `ystar_ustar` does **not** read `ystar` or `ustar`: it re-estimates both
in one likelihood, so `expectations` is its only prerequisite.

Three dependencies are partial: the model runs, but a chart or an option does not.

- `rstar_bonds` estimates r\* without `ystar_ustar`; its Taylor-rule charts need one.
- `rstar_rba`'s Taylor chart reads a completed `rstar_bonds` run and is skipped without one;
  `--employment` needs `ystar_ustar`.
- `ystar`'s `--anchor-phase` needs `expectations`; nothing else in `ystar` does.

## Options most models share

Names vary between models and not all have all of these — check the script header.

| Option | Effect |
|--------|--------|
| `--verbose` / `-v` | Progress and sampler detail |
| `--analyse-only`, `--skip-estimate` | Redraw charts from the saved trace, no sampling |
| `--no-analyse`, `--estimate-only` | Sample and save without charting |
| `--compare` | On `ystar`, `ustar`, `ystar_ustar`: several specifications on one chart |
| `--start`, `--end` | Sample period |
| `--prefix` | Filename stem for the saved trace and observations |

Re-charting from a saved trace is the cheap path and worth trying first: seconds, no re-sampling.

## The summary scripts

`rstar_summary`, `gstar_summary`, `ustar_summary` and `ystar_summary` are not models. Each reads
several saved runs and draws them on one chart.

They share one rule worth knowing first: **any model whose saved trace was not written today is
re-run, which takes minutes and overwrites that model's own output and charts.** `--no-refresh`
charts the saved runs as they stand.

```bash
./run-ystar-summary.sh                 # refresh anything stale, then chart
./run-ystar-summary.sh --no-refresh    # saved runs as they stand
./run-ystar-summary.sh --start 2000Q1
```

`gstar_summary` skips any model with no saved run, but needs at least one.

## GDP nowcasts

Don't run these until about a month before the release. Earlier, almost no indicators for the
target quarter are out: the BVAR declines to nowcast and the bridge and DFM intervals are mostly
prior. `gdp_nowcast_bvar` and `gdp_nowcast_components` are T-0 only.

```bash
uv run python -m src.models.gdp_nowcast_bridge.backtest
```

## No wrapper script

```bash
uv run python -m src.models.rstar_hlw_kalman.run    # canonical HLW, Kalman + ML (failed)
uv run python -m src.models.is_curve.run            # IS curve plotted, not estimated (retired)
uv run python -m src.models.dsge.fa_nk_model        # financial-accelerator DSGE (experimental)
uv run python -m src.models.dsge.fa_nk_wage_model   # FA-NK + sticky wages + Galí unemployment
uv run python -m src.models.dsge.nk_twostar_model   # NK two-star linear probe
uv run python -m src.models.dsge.fa_nk_bayes        # Bayesian re-estimation (--smoke to check)

# How much y* depends on its imposed variances, and on real-time data
uv run python -m src.models.ystar.sigma_sweep --param ratio_g
uv run python -m src.models.ystar.realtime
```

## Diagnostics

Each run writes `run-diagnostics-<prefix>.txt` into its own chart directory. To print MCMC
diagnostics for every saved trace, re-sampling nothing and writing nothing:

```bash
uv run python -m src.models.common.diagnostics_report
uv run python -m src.models.common.diagnostics_report --only ystar
```
