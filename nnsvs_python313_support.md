# NNSVS's Python 3.13 support

## Introduction
NNSVS saw most of its active development around the Python 3.9 era, so running it on a recent Python requires a number of changes to get it to build. The issues and their fixes are listed below.

## Unnecessary build-dependency pin in `pyproject.toml`
`build-system.requires` pins `numpy<1.20.0`, but no wheel for that NumPy version is available for Python 3.13.

### Fix
Simply remove the pin.

## Outdated hydra-core dependency
`antlr4-python3-runtime==4.8`, a dependency of hydra-core 1.1.x, imports `typing.io` internally, but `typing.io` was removed in Python 3.13.

### Fix
Bump `hydra-core` to the 1.3.x line.

## `pkg_resources` import
`pyworld`, `pysptk`, and `nnmnkwii` import `pkg_resources` at runtime, but `pkg_resources` is slated for removal in setuptools 82.0.0; combining it with a newer setuptools results in a `ModuleNotFoundError`.

### Fix
Pin `setuptools<81`. nnsvs itself also used `pkg_resources.resource_filename()` to resolve a path to a packaged resource; this was replaced with a plain `__file__`-based path resolution in anticipation of the future removal.

## Issues stemming from the NumPy 1.x → NumPy 2.0 transition
NumPy 2.0 removed the implicit conversion of non-0-dimensional arrays (including size-1 arrays) to Python scalars. As a result, code that applied the builtin `min()`/`max()` directly to multi-dimensional arrays started raising exceptions. Similarly, some call sites needed explicit conversion or type adjustment when passing scalars.

### Fix
- `nnsvs/train_util.py`: replaced all affected calls with `np.min()`/`np.max()` (explicit NumPy reduction functions)
- `nnsvs/gen.py`, `recipes/_common/scaler_joblib2npy_voc.py`: added explicit `.item()` calls to convert size-1 arrays to scalars
- `nnsvs/dsp.py`: adjusted the type of a value passed into `scipy.signal.butter`

## matplotlib-related issues
A `matplotlib<3.6.0` pin had been added to avoid an incompatibility involving `np.Inf` (a deprecated alias removed in NumPy 2.0), but the referenced issue (#191) does not reproduce on matplotlib 3.11.0. In addition, the naming convention for matplotlib's bundled styles has changed.

### Fix
Remove the `matplotlib<3.6.0` pin. Change `seaborn-whitegrid` to `seaborn-v0_8-whitegrid` in `nnsvs/train_util.py`.

## `typed-ast` build failure
`typed-ast`, a dependency of pysen, fails to build under modern GCC. `typed-ast`'s `Include/asdl.h` defines `typedef enum {false, true} bool;`, but under C23 `false`/`true` are reserved keywords, and this collides with the definition, causing a compile error. Bumping black/mypy/etc. to newer versions might drop the dependency on `typed-ast`, but doing so risks changing the enforced lint rules.

### Fix
Add a Python 3.13 test job to `.github/workflows/ci.yml`, with `lint: false` to explicitly skip the lint step (black/mypy/flake8/isort via pysen).

## ParallelWaveGAN
`parallel_wavegan/layers/pqmf.py` tries to import the old `scipy.signal.kaiser` instead of `scipy.signal.windows.kaiser`. In addition, `setup.py` used to `import pip` to check the pip version, which raises a `ModuleNotFoundError` under `pip install .` in a build-isolated environment.

### Fix
- `parallel_wavegan/layers/pqmf.py`: change `from scipy.signal import kaiser` to `from scipy.signal.windows import kaiser`
- Remove the `import pip`/`distutils.version.LooseVersion`-based check from `setup.py`. Declare the Python version requirement via the standard `python_requires=">=3.7"` argument of `setup()` instead (alternatively, passing `--no-build-isolation` to `pip install` also works).

## HN-UnifiedSourceFilterGAN/SiFiGAN
As with ParallelWaveGAN above, the `import pip` check in `setup.py` causes a `ModuleNotFoundError` under build isolation.

### Fix
Same as above.
