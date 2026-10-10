# Contributing

Thank you for your interest in contributing to `embpy`!

## Local Development Workflow

To set up a local development environment, we recommend using the Github branch to install all required dependencies (including test dependencies).

```bash
# 1. Clone the repository
git clone https://github.com/theislab/embpy.git
cd embpy

# 2. Check out the development branch
git checkout vibe_embpy

# 3. Install in editable mode with development dependencies
pip install -e ".[dev,test,models]"
```

## Running Tests

We use `pytest` for all unit testing. To run the full suite:

```bash
python -m pytest tests/ -v
```

If you are developing a specific model, you can run tests for just that model:
```bash
python -m pytest tests/test_models.py -k "esm2"
```

## CI Testing Matrices

Our GitHub Actions automatically run the test suite against a multi-dimensional matrix:
- **OS:** Ubuntu (Linux), macOS (M1/x86), Windows
- **Python Versions:** 3.11, 3.12, 3.13

Please ensure your changes pass across all combinations before requesting a review. If you add OS-specific code (e.g. `multiprocessing` tweaks for Windows), clearly document it.
