# Quantized Residual Distribution Analysis

Low-bit-depth (b in {1, 2, 3}) speech compression analysis for a subtractively
dithered uniform mid-riser quantizer. The goal is to demonstrate that the
first-order quantized residual, dZ_n = Z_n - Z_{n-1}, concentrates far more
probability mass at zero than the raw quantizer states Z_n, and to quantify the
resulting temporal redundancy via H(Z_n) and H(Z_n | Z_{n-1}).

Lightweight ASR - WINLAB summer project (dithered quantization for ASR-preserving
speech codecs).

## Layout

```
src/
  dither.py       subtractive dither model  f_V(v) = a*Pi + (1-a)*delta
  quantizer.py    mid-riser quantizer, quantization loop, level verification
  entropy.py      marginal + conditional entropy, first-order residuals
  vad.py          frame-energy VAD for silence handling
  data.py         LibriSpeech dev-clean extraction / load / peak-normalize
experiments/
  pmf_comparison.py   Deliverable 3: PMF of Z_n vs dZ_n, b in {1,2,3}
  entropy_sweep.py    Deliverable 4: H(Z), H(Z|Z-1) vs alpha, clip fractions
tests/
  test_sanity.py      IID / constant / AR(1) / quantizer-level checks
notebooks/
  analysis.ipynb      narrative driver importing from src/
```

## Setup

```bash
pip install -r requirements.txt
```

Download `dev-clean.tar.gz` from <https://www.openslr.org/12> and place it in
`data/`. It is gitignored (~337 MB).

## Run

```bash
python -m experiments.pmf_comparison   # Deliverable 3 -> results/figures/
python -m experiments.entropy_sweep    # Deliverable 4 -> results/figures/
pytest tests/                          # sanity checks
```

## Method notes

- **Z_n is the discrete quantizer output** Q(X+V) (bin index k, equivalently
  level C_k), following the project documentation's compression section. The
  subtractive reconstruction Q(X+V)-V is computed but not entropy-coded:
  subtracting the continuous dither would destroy the finite alphabet.
- **Reconstruction levels** C_k = -1 + (k + 0.5) * Delta, verified in
  `tests/test_sanity.py` against the documentation (b=1: +/-1/2; b=2: +/-1/4,
  +/-3/4; b=3: +/-1/8,...,+/-7/8).
- **Conditional entropy** is estimated over ordered pairs via integer
  pair-encoding, not tuple-`np.unique` (which silently miscounts joint symbols).
  Sanity check: IID input gives H(Z_n | Z_{n-1}) ~ H(Z_n); constant input gives 0.
- **Silence** is handled by reporting both all-samples and active-speech
  (25 ms frame-energy VAD) conditions; residual pairs are never formed across
  file or segment boundaries.
- **No-overload accounting**: the clipped-sample fraction is reported per
  (b, alpha) rather than silently saturating overloads.

## The alpha tradeoff

Dither whitens quantization error and reduces harmonic distortion (MSE / MACE /
MSCO all improve as alpha increases), but it decorrelates the quantized output,
so H(Z_n | Z_{n-1}) increases with alpha. Compression efficiency favors small
alpha; ASR robustness / distortion whitening favors larger alpha. 
