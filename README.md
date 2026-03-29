# ML Project Template

A project for preducting short-term power prognosis for loads/generators on the swedish electrical transmission network, **starting with loads**.

---

## Folder Structure

```
├── configs/                  # YAML or JSON config files (hyperparameters, pipeline settings)
├── data/
│   ├── external/             # Third-party data sources, weather (e.g. wind, solar, temperature), network power data, network model
│   ├── interim/              # Intermediate, partially processed data
│   ├── processed/            # Final, model-ready datasets
│   └── raw/                  # Original, immutable source data — never modify
├── docs/                     # Project documentation and design notes
├── logs/                     # Training logs, run histories
├── models/
│   ├── checkpoints/          # In-progress model snapshots during training
│   └── trained/              # Final serialised model artefacts
├── notebooks/
│   ├── exploratory/          # Scratch notebooks for analysis and experimentation
│   └── reports/              # Polished notebooks for presentation and sharing
├── pipelines/                # Orchestration scripts (ETL → features → inference chains)
├── reports/                  # Generated figures, HTML exports, and output artefacts
├── scripts/                  # CLI entrypoints and standalone executable scripts
├── src/
│   ├── data/                 # Data loaders, dataset classes, preprocessing logic
│   ├── evaluation/           # Metrics, validation routines, scoring
│   ├── features/             # Feature engineering pipelines
│   ├── models/               # Model definitions (e.g. PyTorch modules, sklearn wrappers)
│   ├── training/             # Training loops, loss functions, schedulers
│   └── utils/                # Shared helpers (logging, config parsing, etc.)
└── tests/                    # Unit and integration tests — mirrors src/ structure
    ├── data/
    ├── evaluation/
    ├── features/
    ├── models/
    ├── training/
    └── utils/
```

---

## Key Conventions

- **`data/raw/` is immutable.** Never overwrite or edit raw source files. All transformations produce new files in `interim/` or `processed/`.
- **`configs/`** holds all hyperparameters and pipeline settings. No magic numbers in source code.
- **`notebooks/exploratory/`** is for throwaway analysis. Only clean, reproducible work goes in `notebooks/reports/`.
- **`reports/`** holds generated output (figures, HTML). The notebook source that produced it lives in `notebooks/reports/`.
- **`tests/`** mirrors `src/`.  A test file for `src/features/engineer.py` belongs at `tests/features/test_engineer.py`.

---

## Setup

```bash
pip install -r requirements.txt
```

---

## Dependencies

Document your main dependencies in `requirements.txt`. 
