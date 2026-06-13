# Optoelectronic Sensors with Photonic Memory for Trajectory Prediction

This repository contains code and data used to reproduce the numerical trajectory-prediction experiments reported in the manuscript **“Optoelectronic Sensors with Photonic Memory for Trajectory Prediction”**.

The study compares a conventional image sensor with a ZnO-based photonic-memory sensor in a controlled trajectory-prediction task. A circular object moves on a black background along piecewise-linear trajectories. The conventional sensor records memoryless binary frames, while the photonic-memory sensor produces grayscale frames with fading traces of previous object positions.

## Repository structure

```text
photonic-memory-trajectory/
├── README.md
├── requirements.txt
├── DATA_AVAILABILITY.md
├── configs/
│   └── experiment_grid.yaml
├── data/
│   ├── raw/
│   │   ├── zno_potentiation_decay_trace.txt
│   │   └── zno_relaxation_trace.txt
│   └── results/
│       └── experiment_results.csv
├── scripts/
│   ├── generate_motion_frames.py
│   ├── generate_photonic_memory_frames.py
│   ├── add_gaussian_noise.py
│   └── auxiliary_three_object_motion.py
└── notebooks/
    └── reference/
        ├── conventional_single_frame_reference.ipynb
        └── photonic_memory_400s_single_frame_reference.ipynb
```

## Calibration data

The files `data/raw/zno_potentiation_decay_trace.txt` and `data/raw/zno_relaxation_trace.txt` contain raw experimental calibration traces of the ZnO-based optoelectronic synaptic device used to construct the photonic-memory pixel model. Each file is a two-column whitespace-separated text file: the first column is the measured electrical response proportional to device conductance, and the second column is time in seconds. These traces were used to fit the phenomenological potentiation and relaxation functions that define the illumination-history-dependent pixel response in the numerical sensor model.

## Main scripts

### `scripts/generate_motion_frames.py`

Generates the original binary motion frames used in the trajectory-prediction benchmark. The script creates 600 × 600 black-background images with circular white object(s) moving along piecewise-linear trajectories. For the manuscript experiments, it is run with one object, 1 px/frame motion, and random trajectory-switch intervals of 5–40 steps.

The object trajectory is defined by linear segments:

```text
y = kx + b
```

where the parameters are randomly selected from:

```text
(k, b) = (1.0, 0.0), (2.0, 1.0), (-1.0, 0.0), (0.5, -2.0), (3.0, 5.0)
```

The output PNG filenames encode the frame number and object-center coordinates, which are used as ground-truth trajectory labels.

### `scripts/generate_photonic_memory_frames.py`

Converts the original binary motion frames into photonic-memory sensor frames. For each pixel, the script updates an internal conductance state. Illuminated pixels undergo potentiation according to a cubic fit of the experimentally measured ZnO response, while dark pixels relax according to an exponential relaxation model.

The script generates grayscale photonic-memory images in which previous object positions are retained as fading memory traces. The key input parameter is `time_b`, which defines the effective illumination time per frame. In the manuscript experiments, the photonic-memory regimes are:

```text
time_b = 15 s
time_b = 60 s
time_b = 400 s
```

### `scripts/add_gaussian_noise.py`

Adds Gaussian noise to an existing folder of images. It is used to generate noisy versions of both conventional and photonic-memory datasets.

The manuscript uses the following noise regimes:

```text
mean = 50, std = 10
mean = 80, std = 20
```

### `scripts/auxiliary_three_object_motion.py`

Auxiliary trajectory-generation script with three moving circular objects, random direction changes, boundary reflections, and object collisions. It is not the main generator for the manuscript experiments, because the manuscript benchmark uses a single moving object. It is included only as an auxiliary development script.

## Reference notebooks

The `notebooks/reference/` directory contains representative single-condition training runs used during development:

```text
conventional_single_frame_reference.ipynb
photonic_memory_400s_single_frame_reference.ipynb
```

These notebooks include CNN-LSTM training, Optuna hyperparameter optimization, and Comet ML logging. They are kept for transparency. The recommended reproducible workflow is based on the scripts and configuration files.

## Installation

Create a Python environment and install the required dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

The file `requirements.txt` lists the Python packages required for data generation, image processing, model training, hyperparameter optimization, and notebook execution.

## Experimental configuration

The file `configs/experiment_grid.yaml` provides a machine-readable summary of the experimental grid used in the manuscript. It specifies the sensor types, number of input frames, discretization values, noise regimes, photonic-memory time constants, prediction horizon, model type, optimizer, loss function, hyperparameter-optimization method, and evaluation metrics.

## Reproducing the datasets

### Step 1 — Generate original conventional-sensor frames

Run:

```bash
python scripts/generate_motion_frames.py
```

Use the following manuscript settings:

```text
image size: 600
num_frames: 100000
num_balls: 1
pixels_per_frame: 1
min_steps: 5
max_steps: 40
```

Example output folder:

```text
kill_bill/
```

These images correspond to the conventional memoryless sensor frames.

### Step 2 — Generate photonic-memory frames

Run:

```bash
python scripts/generate_photonic_memory_frames.py
```

Use the original frame folder as input:

```text
input folder: kill_bill
```

Run the script three times with:

```text
time_b = 15
time_b = 60
time_b = 400
```

Example output folders:

```text
kill_bill_time:15/
kill_bill_time:60/
kill_bill_time:400/
```

These folders contain photonic-memory sensor frames generated from the same original motion sequence.

### Step 3 — Generate noisy datasets

Run:

```bash
python scripts/add_gaussian_noise.py
```

Apply Gaussian noise to the conventional dataset:

```text
input folder: kill_bill
mean: 50
std: 10
output folder: kill_bill_51_10
```

```text
input folder: kill_bill
mean: 80
std: 20
output folder: kill_bill_80_20
```

Apply the same procedure to each photonic-memory dataset:

```text
kill_bill_time:15
kill_bill_time:60
kill_bill_time:400
```

Example output folders:

```text
kill_bill_time:15_51_10/
kill_bill_time:15_80_20/
kill_bill_time:60_51_10/
kill_bill_time:60_80_20/
kill_bill_time:400_51_10/
kill_bill_time:400_80_20/
```

### Step 4 — Train and evaluate trajectory-prediction models

The generated image folders are used as input datasets for CNN-LSTM trajectory prediction. The number of input frames is varied from 1 to 6. The model predicts the next three object-center coordinates and is evaluated using ATE and RMSE.

The full experimental grid is defined in:

```text
configs/experiment_grid.yaml
```

It includes:

```text
sensor type: conventional, photonic-memory
input frames: 1, 2, 3, 4, 5, 6
discretization: 5, 20
noise: none, Gaussian(50, 10), Gaussian(80, 20)
photonic-memory time_b: 15, 60, 400 s
```

## Experimental results

The file `data/results/experiment_results.csv` contains trajectory-prediction metrics for the evaluated experimental conditions. Each row corresponds to one sensor configuration, including discretization scenario, sensor type (`normal` or `neuro`), number of input frames, photonic-memory speed where applicable, and the resulting ATE and RMSE values.

## Data storage note

The full generated image datasets are not stored in this repository because they are deterministic intermediate outputs and can be regenerated from the released scripts, configuration files, random seeds, and calibration data.
