# Earth-2 MIP (Alpha)

# Guide for running swin with earth2mip

`source set_interactive_vars.sh`: this sets the relevant environment variables for pytorch and for earth2mip.  Two important ones are the path to 73var dataset with q and the Earth2mip\_Model\_Registry. This also activates the conda environment that I have been using for running swin.

`srun -N 1 --ntasks-per-node=4 --gpus-per-node=4 -u -n 4 python -m earth2mip.inference_ensemble config.json`: that is the run script that I use on the interactive queue.  It can be changed for a job script accordingly.

Note: one thing that is in progress: I used to be able to run with more than 1 node.  I've been having some problems as of late: I think it's because of the modulus DistributedManager and SLURM-clusters.  Perhaps I have to do something with export\_DDP\_vars.sh

# Creating a model registry for Swin

The idea for a model registry to have everything necessary to run the model for inference.  This includes the weights, the global\_means.npy, etc.  There is another important file called `metadata.json` that specifies a loader function.  The loader function determines which loading method that earth2mip should use to load the model.  Loader function are specified in `earth2mip/loaders.py` for Swin.

There are 2 swin versions that we have been using: `earth2mip/networks/swin/` and `earth2mip/networks/swin_residual/`.  The relevant loader loads the appropriate one based on the loader specified in the metada.json for the model.



<!-- markdownlint-disable -->
[![Project Status: Active - The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)
[![GitHub](https://img.shields.io/github/license/NVIDIA/earth2mip)](https://github.com/NVIDIA/earth2mip/blob/master/LICENSE.txt)
[![Documentstion](https://img.shields.io/badge/docs-online-green)](https://nvidia.github.io/earth2mip/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
<!-- markdownlint-enable -->

Earth-2 Model Intercomparison Project (MIP) is a Python based AI framework that
enables climate researchers and scientists to explore and experiment with the use of AI
models for weather and climate.
It provides reference workflows for understanding how AI models capture the physics of
the Earth's atmosphere and how they can work with traditional numerical weather
forecasting models.
For instance, the repo provides a uniform interface for running inference using
pre-trained model checkpoints and scoring the skill of such models using certain
standard metrics.
This repository is meant to facilitate the weather and climate community to come up with
good reference baseline of events to test the models against and to use with a variety
of data sources.

## Installation

Earth-2 MIP will be installable on PyPi upon general release.
In the mean time, one can install from source:

```bash
git clone git@github.com:NVIDIA/earth2mip.git

cd earth2mip && pip install .
```

To verify installation run:

```bash
make pytest
```

## Getting Started

The [notebooks](./examples/notebooks/) folder contains reference Jupyter
notebooks for a few common use cases.
These are the best ways to start exploring Earth-2 MIP:

1. [Ensemble Inference](./examples/notebooks/01_ensemble_inference.ipynb) - This
notebook will demonstrate how to perturb the initial conditions to generate ensemble of
inference outputs for a given pre-trained model and generate some standard metrics to
evaluate the skill of the model.

2. [Model Inter Comparison](./examples/notebooks/02_model_comparison.ipynb) - This
notebook will showcase how to generate forecasts from multiple models with Earth-2 MIP
and how to compare their predictions.

### Basic Inference

Earth-2 MIP provides high-level APIs for running inference with AI models.
For example, the following can be used to run Pangu weather using an initial state from
the climate data store (CDS):

```bash
python
>>> import datetime
>>> from earth2mip.networks import get_model
>>> from earth2mip.initial_conditions import cds
>>> from earth2mip.inference_ensemble import run_basic_inference
>>> time_loop  = get_model("e2mip://dlwp", device="cuda:0")
>>> data_source = cds.DataSource(time_loop.in_channel_names)
>>> ds = run_basic_inference(time_loop, n=10, data_source=data_source, time=datetime.datetime(2018, 1, 1))
>>> ds.chunk()
<xarray.DataArray (time: 11, history: 1, channel: 69, lat: 721, lon: 1440)>
dask.array<xarray-<this-array>, shape=(11, 1, 69, 721, 1440), dtype=float32, chunksize=(11, 1, 69, 721, 1440), chunktype=numpy.ndarray>
Coordinates:
  * lon      (lon) float32 0.0 0.25 0.5 0.75 1.0 ... 359.0 359.2 359.5 359.8
  * lat      (lat) float32 90.0 89.75 89.5 89.25 ... -89.25 -89.5 -89.75 -90.0
  * time     (time) datetime64[ns] 2018-01-01 ... 2018-01-03T12:00:00
  * channel  (channel) <U5 'z1000' 'z925' 'z850' 'z700' ... 'u10m' 'v10m' 't2m'
Dimensions without coordinates: history
```

And you can get ACC/RMSE like this:
```
>>> from earth2mip.inference_medium_range import score_deterministic
>>> import numpy as np
>>> scores = score_deterministic(time_loop,
    data_source=data_source,
    n=10,
    initial_times=[datetime.datetime(2018, 1, 1)],
    # fill in zeros for time-mean, will typically be grabbed from data.
    time_mean=np.zeros((7, 721, 1440))
)
>>> scores
<xarray.Dataset>
Dimensions:        (lead_time: 11, channel: 7, initial_time: 1)
Coordinates:
  * lead_time      (lead_time) timedelta64[ns] 0 days 00:00:00 ... 5 days 00:...
  * channel        (channel) <U5 't850' 'z1000' 'z700' ... 'z300' 'tcwv' 't2m'
Dimensions without coordinates: initial_time
Data variables:
    acc            (lead_time, channel) float64 1.0 1.0 1.0 ... 0.9686 0.9999
    rmse           (lead_time, channel) float64 0.0 2.469e-05 0.0 ... 7.07 2.998
    initial_times  (initial_time) datetime64[ns] 2018-01-01
>>> scores.rmse.sel(channel='z500')
<xarray.DataArray 'rmse' (lead_time: 11)>
array([  0.        , 150.83014446, 212.07880612, 304.98592282,
       381.36510987, 453.31516952, 506.01464974, 537.11092269,
       564.79603347, 557.22871627, 586.44691243])
Coordinates:
  * lead_time  (lead_time) timedelta64[ns] 0 days 00:00:00 ... 5 days 00:00:00
    channel    <U5 'z500'
```

### Supported Models

These notebooks illustrate how-to-use with a few models and this can serve as reference
to bring in your own checkpoint as long as it's compatible. There may be additional work
to make it compatible with Earth-2 MIP.
Earth-2 MIP leverages the model zoo in [Modulus](https://github.com/NVIDIA/modulus) to
provide a reference set of base-line models.
The goal is to enable to community to grow this collection of models as shown in the
table below.

<!-- markdownlint-disable -->
| ID | Model | Architecture | Type | Reference | Source | Size |
|:-----:|:-----:|:-------------------------------------------:|:--------------:|:---------:|:-------:|:---:|
| fcn | FourCastNet | Adaptive Fourier Neural Operator  | global weather | [Arxiv](https://arxiv.org/abs/2202.11214)   | [modulus](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/modulus/models/modulus_fcn) | 300Mb |
| dlwp |  Deep Learning Weather Prediction  |  Convolutional Encoder-Decoder | global weather |   [AGU](https://doi.org/10.1029/2020MS002109)   | [modulus](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/modulus/models/modulus_dlwp_cubesphere) |  50Mb |
| pangu | Pangu Weather (Hierarchical 6 + 24 hr)  |  Vision Transformer | global weather |  [Nature](https://doi.org/10.1038/s41586-023-06185-3) | onnx | 2Gb |
| pangu_6 | Pangu Weather 6hr Model  |  Vision Transformer | global weather |  [Nature](https://doi.org/10.1038/s41586-023-06185-3) | onnx | 1Gb |
| pangu_24 | Pangu Weather 24hr Model |  Vision Transformer | global weather |  [Nature](https://doi.org/10.1038/s41586-023-06185-3) | onnx | 1Gb |
| fcnv2_sm |  FourCastNet v2 | Spherical Harmonics Fourier Neural Operator | global weather |  [Arxiv](https://arxiv.org/abs/2306.03838)  | [modulus](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/modulus/models/modulus_fcnv2_sm) | 3.5Gb |
| graphcast |  Graphcast, 37 levels, 0.25 deg | Graph neural network | global weather |  [Arxiv](https://arxiv.org/abs/2212.12794)  | [github](https://github.com/google-deepmind/graphcast) | 145MB |
| graphcast_small |  Graphcast, 13 levels, 1 deg | Graph neural network | global weather |  [Arxiv](https://arxiv.org/abs/2212.12794)  | [github](https://github.com/google-deepmind/graphcast) | 144MB |
| graphcast_operational |  Graphcast, 13 levels, 0.25 deg| Graph neural network | global weather |  [Arxiv](https://arxiv.org/abs/2212.12794)  | [github](https://github.com/google-deepmind/graphcast) | 144MB |
| precipitation_afno | FourCastNet Precipitation | Adaptive Fourier Neural Operator  | diagnostic | [Arxiv](https://arxiv.org/abs/2202.11214)   | [modulus](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/modulus/models/modulus_diagnostics) | 300Mb |
| climatenet | ClimateNet Segmentation Model | Convolutional Neural Network | diagnostic | [GMD](https://doi.org/10.5194/gmd-14-107-2021)   | [modulus](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/modulus/models/modulus_diagnostics) | 2Mb |
<!-- markdownlint-enable -->

\* = coming soon

We want to integrate your model into the scoreboard to show the community!
The best way to do this is via [NVIDIA Modulus](https://github.com/NVIDIA/modulus).
You can contribute your model (both the training code as well as model checkpoint) and
we can ensure that it is maintained as part of the reference set.

## Contributing

Earth-2 MIP is an open source collaboration and its success is rooted in community
contribution to further the field.
Thank you for contributing to the project so others can build on your contribution.
For guidance on making a contribution to Earth-2 MIP, please refer to the
[contributing guidelines](./CONTRIBUTING.md).

## More About Earth-2 MIP

This work is inspired to facilitate similar engagements between teams here at
NVIDIA - the ML experts developing new models and the domain experts in Climate science
evaluating the skill of such models.
For instance, often necessary input data such as normalization constants and
hyperparameter values are not packaged alongside the model weights.
Every model typically implements a slightly different interface. Scoring routines are
specific to the model being scored and may not be consistent across groups.

Earth-2 MIP addresses  these challenges and bridges the gap between the domain experts
who most often are assessing ML models, and the ML experts producing them.
Compared to other projects in this space, Earth-2 MIP focuses on scoring models
on-the-fly.
It has python APIs suitable for rapid iteration in a jupyter book, CLIs for scoring
models distributed over many GPUs, and a flexible
plugin framework that allows anyone to use their own ML models.
More importantly Earth-2 MIP aspires to facilitate exploration and collaboration within
the climate research community to evaluate the potential of AI models in climate and
weather simulations.

Please see the [documentation page](https://nvidia.github.io/earth2mip/) for in depth
information about Earth-2 MIP, functionality, APIs, etc.

## Communication

- Github Discussions: Discuss new ideas, model integration, support etc.
- GitHub Issues: Bug reports, feature requests, install issues, etc.

## License

Earth-2 MIP is provided under the Apache License 2.0, please see
[LICENSE.txt](./LICENSE.txt) for full license text.

### Additional Resources

- [Earth-2 Website](https://www.nvidia.com/en-us/high-performance-computing/earth-2/)
- [NVIDIA Modulus](https://github.com/NVIDIA/modulus)
