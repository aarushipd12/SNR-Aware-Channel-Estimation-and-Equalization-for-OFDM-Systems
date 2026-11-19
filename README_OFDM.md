# SNR-Aware Hybrid Channel Estimation and Equalization for OFDM Systems

**Comb-pilot LS estimation · Delay-domain denoising · Doppler-aware CE-BEM · End-to-end audio transmission**

This project implements and evaluates an **Orthogonal Frequency Division Multiplexing (OFDM)** link over multipath Rayleigh fading channels. It compares pilot-based channel estimation and equalization methods under both **block fading (static within an OFDM symbol)** and **time-varying Doppler channels**, using bit error rate (BER), channel-estimation mean squared error (MSE), and reconstructed audio quality as performance metrics.

The Jupyter notebook contains the complete simulation, plots, and a demonstration that transmits an audio clip through the simulated OFDM link. The accompanying PDF explains the system model, methods, and results.

## Repository contents

```text
.
├── OFDM_channel_estimation_and_equalization_with_audio_demo.ipynb
├── Presentation.pdf
└── README.md
```

- **`OFDM_channel_estimation_and_equalization_with_audio_demo.ipynb`** — OFDM transmitter/receiver, channel simulation, estimation, filtering, equalization, BER/MSE evaluation, forward error correction, and audio demonstrations.
- **`Presentation.pdf`** — Project presentation with methodology, visualizations, comparisons, and conclusions.

## Objectives

- Implement an OFDM transmitter and receiver using Gray-coded QPSK modulation (with a 16-QAM option).
- Estimate the frequency response of a Rayleigh fading channel using **least-squares (LS) estimation** at comb pilots.
- Compare **linear** and **cubic-spline** interpolation between pilot positions.
- Reduce estimation noise using **delay-domain (DFT-based) filtering**.
- Benchmark channel estimation and equalization against **perfect channel state information (CSI)**.
- Evaluate the influence of **SNR, delay-window length, and Doppler spread** on reception quality.
- Extend the receiver to time-varying channels using a **complex exponential basis expansion model (CE-BEM)** and **banded MMSE equalization**.
- Demonstrate the practical impact through reconstructed audio, optional **Hamming(7,4) coding and interleaving**, and spectrograms.

## OFDM system configuration

| Parameter | Default value |
| --- | --- |
| FFT size / subcarriers | 64 |
| Cyclic prefix | 16 samples |
| Modulation | Gray-coded QPSK |
| Channel | 4-tap Rayleigh multipath fading |
| Tap-power profile | Exponential, normalized to unit total power |
| Comb-pilot spacing | Every 4 subcarriers, plus subcarrier 63 |
| Pilot / data subcarriers | 17 / 47 |
| Delay-domain window | 8 bins |
| Single-SNR experiment | 20 dB, 200 OFDM symbols |
| Audio sample rate | 8 kHz, mono |
| Audio coding | 8-bit μ-law, μ = 255 |
| Doppler parameter (mobile demo) | ε = 0.2 |
| CE-BEM band half-width | Q = 2 |

The static-channel simulation assumes the channel remains constant over each OFDM symbol but changes independently from one symbol to the next. The Doppler extension explicitly allows the channel taps to vary **within** an OFDM symbol, generating inter-carrier interference (ICI).

## Methodology

### 1. OFDM transmission and channel model

The transmitter maps payload bits to QPSK symbols, inserts known comb pilots, performs an IFFT, and adds a cyclic prefix. The simulated Rayleigh channel introduces multipath fading and additive noise. The receiver removes the cyclic prefix, applies an FFT, estimates the channel, equalizes the data subcarriers, and demaps the recovered symbols.

For the block-fading channel, each subcarrier follows:

$$
Y[k] = H[k]X[k] + W[k].
$$

### 2. Pilot-based LS channel estimation

At pilot subcarrier $k_p$, the receiver uses the known transmitted pilot $X_p$:

$$
\hat{H}[k_p] = \frac{Y[k_p]}{X_p}.
$$

The pilot estimates are interpolated across all subcarriers using either **linear interpolation** or a **cubic spline**.

### 3. Delay-domain denoising

Because the channel has relatively few significant multipath taps, its impulse response occupies only a small part of the delay domain. The estimator therefore:

1. Applies an **IDFT** to the interpolated channel estimate.
2. Retains the first **8 delay bins**, setting the remainder to zero.
3. Applies a **DFT** to obtain a denoised frequency-domain channel estimate.

For the simulated channel, the chosen window satisfies the design constraint:

$$
L \le W \le \min(\mathrm{CP}, N/P).
$$

### 4. Equalization and comparison

The notebook evaluates zero-forcing (ZF) and minimum mean squared error (MMSE) equalization, together with perfect-CSI reference cases. For one-tap equalization:

$$
\hat X_{\mathrm{ZF}}[k] = \frac{Y[k]}{\hat H[k]},
\qquad
\hat X_{\mathrm{MMSE}}[k] = \frac{\hat H^*[k]Y[k]}{|\hat H[k]|^2 + \sigma^2}.
$$

With hard-decision QPSK, one-tap ZF and MMSE applied to the **same channel estimate** yield the same bit decisions; the static audio comparison also varies the estimator.

### 5. Audio transmission and forward error correction

The notebook converts a WAV file to mono, resamples it to **8 kHz**, peak-normalizes it, and applies **8-bit μ-law companding**. The resulting bitstream is transmitted using OFDM and reconstructed at each receiver.

Optional **Hamming(7,4) coding with interleaving** helps correct bit errors introduced by the fading channel. Audio quality is evaluated using waveform SNR relative to the original clip, so the metric includes both channel errors and μ-law quantization distortion.

### 6. Doppler-aware channel estimation and equalization

When the channel changes within an OFDM symbol, subcarriers interfere and the one-tap static-channel model becomes inadequate. The mobile extension uses:

- **CE-BEM** to represent time-varying channel taps with a small number of Doppler basis coefficients.
- **Clustered pilots with guard nulls** to estimate the model coefficients.
- **Banded LDLᴴ-based MMSE equalization** to mitigate coupling between neighboring subcarriers.
- A **genie/full-MMSE reference**, which uses the true time-varying channel for benchmarking.

## Results

### Static Rayleigh channel — 20 dB

The single-SNR comparison in the presentation reports:

| Channel estimator | BER | Channel MSE |
| --- | ---: | ---: |
| Perfect CSI | 0.00543 | 0 |
| LS + linear interpolation | 0.01059 | 9.78 × 10⁻³ |
| LS + cubic spline | 0.01027 | 9.11 × 10⁻³ |
| LS linear + delay filter | 0.00856 | 5.56 × 10⁻³ |
| LS spline + delay filter | **0.00809** | **4.69 × 10⁻³** |

Delay filtering reduces channel-estimation MSE by approximately **43%** for linear interpolation and **49%** for spline interpolation in this experiment, with corresponding BER improvements.

### Static audio demo — 20 dB

| Receiver | BER | Audio SNR |
| --- | ---: | ---: |
| No equalization | 0.50908 | −2.4 dB |
| Comb-linear LS + ZF | 0.00946 | 13.5 dB |
| DFT-denoised linear + MMSE | 0.00733 | 14.6 dB |
| DFT-denoised spline + MMSE | **0.00717** | **14.8 dB** |
| Ideal CSI + ZF | 0.00492 | 16.4 dB |

### Doppler/mobile audio demo — 30 dB, ε = 0.2

| Receiver | BER | Audio SNR |
| --- | ---: | ---: |
| No equalization | 0.49476 | −2.4 dB |
| Static LS + delay filter, one-tap | 0.09902 | 4.0 dB |
| CE-BEM diagonal + one-tap ZF | 0.02387 | 8.9 dB |
| **CE-BEM + banded LDLᴴ MMSE** | **0.01341** | **11.7 dB** |
| Genie full MMSE (true channel) | 0.00057 | 25.6 dB |

**Key takeaway:** Delay-domain filtering improves channel estimation in the block-fading setting, while Doppler-induced ICI requires a time-varying channel model and multi-subcarrier equalization. The practical receivers do not reach the ideal-CSI benchmarks.

*These are results shown in `Presentation.pdf` for specific simulation settings and random channel/noise realizations. Re-running experiments or changing parameters may produce different values.*

## Getting started

### Requirements

- Python 3.9+ (recommended)
- Jupyter Notebook or JupyterLab
- NumPy
- SciPy
- Matplotlib
- IPython (for inline audio playback)

Install the dependencies:

```bash
python -m pip install numpy scipy matplotlib ipython jupyter
```

### Run the notebook

1. Clone or download the repository.
2. Open a terminal in the repository directory.
3. Launch Jupyter:

   ```bash
   jupyter notebook
   ```

4. Open `OFDM_channel_estimation_and_equalization_with_audio_demo.ipynb`.
5. Select **Run All** to execute the static OFDM experiments, audio demonstrations, and Doppler analysis in order.

The notebook can also be opened in Google Colab; audio output paths are relative to the notebook's current working directory.

### Use your own audio

In the audio configuration cell (Section 10), set:

```python
WAV_PATH = "input.wav"   # path to a local WAV file
OUT_DIR = "audio_out"
FS = 8000
MAX_SECONDS = 3
```

Place `input.wav` in the notebook's working directory, or change `WAV_PATH` to the actual file path. **If the WAV file is missing, the notebook automatically generates a synthetic test clip**, so an external audio file is not required.

The notebook saves reconstructed `.wav` files in `audio_out/` and displays audio players and plots inline. The default comparison processes up to **3 seconds** of audio.

### Experiment with the parameters

You can modify these notebook settings and rerun from the top:

```python
modulation_type = "QPSK"  # or "16QAM"
PILOT_SPACING = 4
WIN_LEN = 8
snr_db = 20
```

For the audio demonstrations, change `DEMO_SNR` or enable `USE_FEC = True` in the respective run cell. For the mobile experiment, explore `EPS` and the CE-BEM parameter `Q`, while respecting the notebook's configuration checks.

## Evaluation and visualizations

The notebook includes:

- QPSK constellation plots before and after equalization.
- True, LS-interpolated, and delay-filtered channel responses.
- Delay-domain response and filtering-window visualizations.
- **BER and channel MSE versus SNR** and the impact of delay-window length.
- Audio waveforms, listening comparisons, and spectrograms.
- Audio BER/SNR comparisons with and without Hamming(7,4).
- **Mobile BER versus SNR and normalized Doppler**, including CE-BEM receiver benchmarks.

## Limitations

- The initial model assumes independent block fading between OFDM symbols rather than temporal channel tracking.
- Delay-domain filtering suppresses estimation noise but cannot eliminate interpolation error or Doppler-induced ICI.
- CE-BEM approximates channel variation, so its performance remains below the true-channel MMSE reference.
- Clustered pilots and guard nulls consume spectral resources, and limited pilot observations make mobile estimation sensitive to noise.
- Results are based on simulation, not over-the-air hardware measurements.

## Presentation

See **[`Presentation.pdf`](Presentation.pdf)** for system diagrams, algorithm explanations, performance plots, and the audio demonstration comparisons.

---

**Project focus:** DSP and digital communication techniques for robust OFDM reception under frequency-selective fading and time-varying channels.
