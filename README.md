# STFT Magnitude/Phase and STOI Analysis

Comprehensive experiments on the role of STFT magnitude and phase in speech intelligibility, signal reconstruction from magnitude only, phase scrambling and recovery, and the effects of additive noise and band-pass filtering on STOI.

- **Audio Source**: 10-second excerpt from *"Lecture 1, Introduction | MIT RES.6.007 Signals and Systems, Spring 2011"* (`voice_rec.wav`)
- **Interactive Report**: [View Notebook Online](https://shreeshbhat04-ctrl.github.io/Assignment/)
- **Additional Explorations**: See [`miscilleaneous/test.ipynb`](miscilleaneous/test.ipynb) for detailed parameter sweeps (window length, overlap, Griffin-Lim iterations, Fast Griffin-Lim Algorithm (FGLA) vs. other recovery methods, filter families, and band cutoffs).

---

## Answers and Analysis

### 1. Difference Between the Original Signal and `sig1`

#### Analysis and Differences
- **Amplitude Envelope**: The amplitude envelopes of the original signal and `sig1` match closely; speech timing, rhythm, and tone are preserved over time because the STFT magnitude ($MG$), which contains energy distribution across time and frequency, was completely retained.
- **Waveform Discrepancy**: The time-domain waveforms do not align sample-by-sample:
  - Correlation with original: **0.043**
  - SI-SDR: **-27.3 dB**
- **Intelligibility (STOI)**: Yields **0.8419**, indicating high speech intelligibility despite zero phase.
- **Amplitude & Energy**:
  - Peak amplitude is substantially lower in `sig1` (0.042 vs. 0.173 in the original).
  - RMS amplitude is also lower (0.0026 vs. 0.0125 in the original).

#### Reasoning
- STOI evaluates the correlation of spectral envelopes across short-time blocks (~26 ms). Because the envelope is governed by STFT magnitude, STOI is largely insensitive to phase distortion.
- In contrast, waveform-level metrics (such as SI-SDR) and human auditory perception clearly capture the missing phase structure, causing `sig1` to sound distinctly unnatural and robotic.

---

### 2. Is It Possible to Obtain a Real-Valued `sig2`?

- **Answer**: **No** for an arbitrary random phase matrix $\text{PH1}$, but **yes** if $\text{PH1}$ satisfies conjugate symmetry.
- **Reasoning**:
  - A real-valued discrete time-domain signal requires a conjugate-symmetric frequency spectrum:
    $$X[N - k] = X^*[k]$$
  - Since $MG$ is the magnitude of a real signal, it is already symmetric (even) in frequency. Therefore, the condition reduces to the phase being an odd function of frequency ($\angle X[N-k] = -\angle X[k]$).
  - An arbitrary random phase assignment breaks this conjugate symmetry, preventing imaginary components across frequency bins and overlapping frames from canceling out during inverse STFT (overlap-add).

---

### 3. Difference Between the Original Signal and `sig2`

#### Analysis and Differences
- Similar to `sig1`, the broad amplitude envelope matches the original speech, but the time-domain waveform differs substantially:
  - Correlation with original: **-0.0685**
  - SI-SDR: **-23.27 dB**
- **STOI**: Remains high at **0.86**, confirming that the speech envelope remains intelligible.
- **Auditory Quality**: `sig2` sounds muffled and reverberant, similar to talking underwater.

```python
def si_sdr(ref, est, eps=1e-12):
    ref = ref - ref.mean()
    est = est - est.mean()
    alpha = np.dot(est, ref) / (np.dot(ref, ref) + eps)
    target = alpha * ref
    noise = est - target
    return 10 * np.log10((np.sum(target**2) + eps) / (np.sum(noise**2) + eps))
```

#### Reasoning
- Retaining $MG$ preserves the spectral energy envelope and linguistic content, explaining the high STOI score.
- Because $\text{PH1}$ is random, the relative phase timing among frequency bins within each frame is scrambled. Moreover, phase values do not align across successive overlapping frames, yielding an inconsistent STFT and the characteristic underwater sound.

---

### 4. Difference Between `sig1` and `sig2`

#### Analysis and Differences
- The two signals are essentially uncorrelated at the waveform level:
  - Correlation: **0.009**
  - SI-SDR: **-40.9 dB**
- **Loudness & Energy**: `sig2` is louder than `sig1`:
  - RMS: **0.0064** (`sig2`) vs. **0.0026** (`sig1`) (a difference of ~7.8 dB).
  - Re-synthesizing and re-analyzing the signals retains **26.4%** of the original magnitude energy for `sig2`, compared to only **4.2%** for `sig1`.
- **Envelope Similarity**: Computing STOI between `sig2` and `sig1` (using `sig1` as reference) yields **0.784**, reflecting strong envelope similarity between both signals rather than speech intelligibility.

#### Reasoning
- Both signals are synthesized from the identical magnitude matrix $MG$, explaining their similar envelope profiles and moderate mutual STOI.
- However, their phase distributions are completely distinct (zero phase vs. random phase), making their resulting time-domain waveforms mutually uncorrelated.

![Comparison of original, sig1, and sig2 waveforms and spectrograms](image.png)

---

### 5. Pattern in `PH1` That Ensures a Real-Valued `sig2`

To guarantee a real-valued time-domain signal, the complex spectrum must satisfy conjugate symmetry:
$$X[N - k] = X^*[k]$$

This imposes two conditions on $\text{PH1}$:
1. **Odd Symmetry Across Frequency**:
   $$\text{PH1}[N - k] = -\text{PH1}[k] \quad \text{for } k = 1, \dots, \frac{N}{2} - 1$$
   *(Bins where $MG = 0$ do not contribute and phase can be arbitrary).*
2. **Real-Valued DC and Nyquist Bins**:
   $$\text{PH1}[0] \in \{0, \pi\} \quad \text{and} \quad \text{PH1}\left[\frac{N}{2}\right] \in \{0, \pi\}$$

#### Numerical Verification (`miscilleaneous/test.ipynb`)
- **Arbitrary Random $\text{PH1}$**: $\max|\text{imag}| = 5.93 \times 10^{-2}$ vs. $\max|\text{real}| = 6.37 \times 10^{-2}$ $\rightarrow$ signal is complex.
- **Odd-Symmetric $\text{PH1}$ (with DC & Nyquist $= 0$)**: $\max|\text{imag}| = 1.24 \times 10^{-17}$ $\rightarrow$ signal is real to machine precision.

---

### 6. Number of Valid Real-Valued Phase Patterns

For a frame size of $N = 512$:
- There are $N/2 - 1 = 255$ free positive-frequency phase bins ($k = 1, \dots, 255$).
- Each free bin can be assigned any phase value in $[0, 2\pi)$, which uniquely defines the negative frequency bins by odd symmetry.
- The DC ($k = 0$) and Nyquist ($k = 256$) bins each have 2 valid choices ($0$ or $\pi$), contributing $2 \times 2 = 4$ combinations per frame.
- With $F = 1795$ independent frames across the signal:
  - **Continuous Case (Real Numbers)**: There are **infinitely many** valid phase patterns (a continuum of dimension 255 per frame, or $255 \times F$ dimensions total).
  - **Quantized Case ($L$ levels)**: If phase is quantized to $L$ discrete levels, there are $4 \cdot L^{255}$ valid patterns per frame, yielding $(4 \cdot L^{255})^F$ patterns for the complete signal.

---

### 7. Band-Pass Filter Design Choice and Rationale

Multiple band-pass filter topologies were evaluated (FIR Hamming, Chebyshev Type I & II, Elliptic, and Butterworth):

| Metric | White Noise | Babble Noise |
|---|---|---|
| **SNR Gain at 100–4000 Hz** (Butterworth) | **+4.2 dB** | **+0.06 dB** |
| **STOI Change at 100–4000 Hz** | **+0.0007** | **-0.0012** |
| **SNR Gain at 300–3400 Hz** (Butterworth) | +4.4 dB | +1.4 dB |
| **STOI Change at 300–3400 Hz** | -0.049 | -0.013 |

#### Selection Rationale
- **Chosen Filter**: **4th-order Butterworth, zero-phase (`sosfiltfilt`), 100–4000 Hz**.
- **Justification**:
  - Provides **+4.2 dB** SNR gain on wideband white noise with virtually zero STOI degradation ($\Delta\text{STOI} \approx 0$).
  - Offers a maximally flat passband (no ripple) and zero phase distortion through forward-backward filtering.
  - While narrower passbands (such as telephone band 300–3400 Hz) or Chebyshev II achieve slightly higher broadband SNR gains, they truncate essential speech formant frequencies, resulting in measurable STOI loss. Hence, SNR alone is not the sole criterion.

![Filter response and STOI vs SNR](image-1.png)

---

### 8. Comments on STOI Scores Across Noise Conditions

#### STOI vs. Clean Reference (Unfiltered)
| Noise Type | -2 dB | 0 dB | 2 dB | 5 dB | 10 dB |
|---|---|---|---|---|---|
| **White** | 0.690 | 0.732 | 0.773 | 0.826 | 0.893 |
| **Machine Gun** | 0.761 | 0.789 | 0.816 | 0.854 | 0.909 |
| **Babble** | 0.579 | 0.627 | 0.674 | 0.741 | 0.837 |

#### Observations
- **Effect of Filtering**: Band-pass filtering leaves STOI essentially unchanged across all 15 test conditions (variations stay within **-0.0014 to +0.0011**):
  - Babble: -0.0011 to -0.0014
  - Machine gun: -0.0006 to +0.0001
  - White noise: +0.0011 to -0.0013
- **Input-Independent SNR Gain**: The SNR improvement from band-pass filtering depends on the noise spectral distribution, not the input SNR level:
  - White noise: **+4.24 dB**
  - Machine gun: **+1.12 dB**
  - Babble: **+0.06 dB**
- **Band-Pass Filtering vs. STOI**: Linear band-pass filtering is an ineffective mechanism for improving speech intelligibility (STOI). While it attenuates out-of-band noise energy (boosting SNR for wideband noise), STOI exclusively evaluates intelligibility within the active speech bands. Cutting into speech frequencies degrades STOI. The 100–4000 Hz Butterworth band was selected because it preserves speech intelligibility ($\Delta\text{STOI} \approx 0$) while eliminating out-of-band white noise.

---

### 9. Phase Scrambling, Recovery, and Noise Analysis (Tasks 5 & 6)

- **Scrambled Signal**: Magnitude $MG$ combined with random odd phase (with DC and Nyquist set to 0) to yield a real-valued inverse signal.
  - Metrics: STOI = **0.8607**, SI-SDR = **-27.1 dB** (audibly robotic).
- **Fast Griffin-Lim Recovery** (300 iterations, momentum = 0.99):
  - Recovering from the scrambled signal's own magnitude yields STOI = **0.8607** (no intelligibility gain). Re-analyzing scrambled audio produces an inconsistent STFT magnitude due to overlap-add phase cancellation.
  - **Oracle Recovery** (using original true magnitude $MG$): Restores high speech intelligibility (STOI = **0.9986**), though the phase converges to an estimate different from the original, leaving waveform-level SI-SDR at -25.2 dB.

```python
def recover(x, sr, nfft, hop, n_iter=300, momentum=0.99, seed=0, mag=None):
    if mag is None:
        mag = np.abs(librosa.stft(x, n_fft=nfft, hop_length=hop, win_length=nfft, window="hann"))
    rec = librosa.griffinlim(
        mag,
        n_iter=n_iter,
        momentum=momentum,
        hop_length=hop,
        win_length=nfft,
        window="hann",
        length=len(x),
        rng=seed,
    )
    return rec

# Magnitude reconstructed from scrambled audio
sig_rec = recover(sig_scram, sr, n_fft, hop_len)

# Magnitude taken from original clean STFT (oracle)
sig_rec_oracle = recover(sig_scram, sr, n_fft, hop_len, mag=MG)
```

```text
STOI scrambled        : 0.8607286214091554
STOI recovered        : 0.8606968074751862
STOI recovered oracle : 0.9985975227711728
```

- **Noise Effects**:
  - Across all 3 noise types and 5 SNR levels, STOI monotonically increases with higher SNR.
  - Phase recovery algorithms do not perform denoising; they only reconstruct phase consistency. An oracle recovery on noisy audio cannot exceed the intrinsic intelligibility of that noisy audio.

---

## Repository Contents

```text
.
├── assignment.ipynb              # Main analysis notebook
├── README.md                     # Project documentation
├── requirements.txt              # Python dependencies
│
├── docs/
│   └── index.html                # HTML version of the notebook / report
│
├── miscilleaneous/               # Additional experiments and sweep data
│   ├── test.ipynb
│   ├── nfft_sweep.csv
│   └── recovery_results.csv
│
├── bandpass_outputs/             # Band-pass filter results and logs
│   └── bandpass.csv
│
├── outputs/                      # Generated noisy audio files
│   ├── babble_-2dB.wav
│   ├── babble_0dB.wav
│   ├── babble_2dB.wav
│   ├── babble_5dB.wav
│   ├── babble_10dB.wav
│   ├── machinegun_-2dB.wav
│   ├── machinegun_0dB.wav
│   ├── machinegun_2dB.wav
│   ├── machinegun_5dB.wav
│   ├── machinegun_10dB.wav
│   ├── white_-2dB.wav
│   ├── white_0dB.wav
│   ├── white_2dB.wav
│   ├── white_5dB.wav
│   └── white_10dB.wav
│
├── babble.wav                    # Babble noise source
├── machinegun.wav                # Machine-gun noise source
├── white.wav                     # White noise source
├── voice_rec.wav                 # Voice recording source
├── image.png                     # Analysis spectrogram / waveform comparison
└── image-1.png                   # Filter frequency response and STOI plot
```

---

## Setup

Install the required Python dependencies:

```bash
pip install -r requirements.txt
```

**Core Libraries**: `numpy`, `scipy`, `matplotlib`, `pandas`, `librosa`, `soundfile`, `pystoi`.
- Tested with Python 3.13, `librosa` 1.0.0, and `pystoi` 0.4.1.

Noise sources are from the NOISEX-92 dataset ([available here](https://github.com/speechdnn/Noises/tree/master/NoiseX-92)): `white.wav`, `machinegun.wav`, and `babble.wav`.

---

## How to Run

1. Open `assignment.ipynb` in Jupyter Notebook or VS Code.
2. Run all cells sequentially from top to bottom.
3. Generated audio files are saved to `outputs/` and filter results to `bandpass_outputs/`.
4. *Estimated runtime*: ~15 minutes. Random seeds are fixed (`seed = 0`) for full reproducibility.

---

## Experimental Settings

| Parameter | Value |
|---|---|
| **Recording** | 10.42 s (~10 s), mono, sampled at 22,050 Hz (`voice_rec.wav`) |
| **STFT** | Hann window, `n_fft = 512`, `hop_length = 128` (75% overlap) |
| **Random Seed** | `0` (`np.random.default_rng(0)`) |
| **Noise Mixing** | Random segment of each NOISEX file, resampled to 22,050 Hz, scaled to target SNR (-2, 0, 2, 5, 10 dB) |
| **STOI** | `pystoi.stoi`, standard (not extended), reference = original clean recording unless stated |
| **Phase Recovery** | Fast Griffin-Lim, 300 iterations, momentum = 0.99, seed = 0 |
| **Band-Pass Filter** | 4th-order Butterworth, zero-phase (`sosfiltfilt`), 100–4000 Hz |

---

## Task Coverage

| Task Item | Notebook Function / Cell |
|---|---|
| 2: STFT magnitude and phase plots, `sig1` | `fnc_2to5` |
| 3: `sig2` with random phase `PH1`, real-valuedness | `cont_fnc_2to5` |
| 4: Pattern in `PH1` ensuring real-valued `sig2` | `cont_fnc_2to5` |
| 5 & 6: Odd random phase, recovery, noise conditions, STOI | `fnc_randnoise` |

---

## Key Findings

- **`sig1` (Zero Phase)**: Correlation with original = 0.0432, SI-SDR = -27.3 dB, STOI = 0.8420.
- **`sig2` (Random Phase)**: Correlation with original = -0.0685, SI-SDR = -23.27 dB, STOI = 0.8607.
- **Real-Valued Inversion**: An arbitrary random $\text{PH1}$ yields a complex signal ($\max|\text{imag}| = 5.93 \times 10^{-2}$ vs. $\max|\text{real}| = 6.37 \times 10^{-2}$). An odd-symmetric $\text{PH1}$ with DC and Nyquist set to 0 yields a real-valued signal ($\max|\text{imag}| = 1.24 \times 10^{-17}$).
- **Phase Recovery**: Reconstructing from the scrambled signal alone yields STOI = 0.8607 (no intelligibility improvement); oracle reconstruction using the true clean magnitude achieves STOI = 0.9986 (SI-SDR = -25.19 dB due to phase ambiguity).
- **Noise Performance**: STOI increases monotonically with SNR across all noise types (detailed results in [`miscilleaneous/recovery_results.csv`](miscilleaneous/recovery_results.csv)).
- **Band-Pass Filtering**: Achieves SNR gains of +4.24 dB (white), +1.12 dB (machine gun), and +0.06 dB (babble), while keeping STOI changes within $\pm 0.0014$.

---

## Assumptions and Interpretations

- **Inverse FT**: Interpreted as the inverse Short-Time Fourier Transform (per-frame IFFT followed by overlap-add).
- **Recover the Original**: Interpreted as phase retrieval using the Fast Griffin-Lim Algorithm (evaluated both from the scrambled signal's magnitude and from the true clean magnitude oracle).
- **Repeat for Each Noise**: Interpreted as generating `sig1` and `sig2`, applying phase scrambling, and running phase recovery across each noise condition and SNR.

---

## Audio Examples

| File | Description |
|---|---|
| `original.wav` / `voice_rec.wav` | Clean speech recording |
| `sig1_zero_phase.wav` | Reconstructed from magnitude only with zero phase |
| `sig2_random_phase.wav` | Reconstructed with arbitrary random phase $\text{PH1}$ |
| `scrambled.wav`, `recovered.wav`, `recovered_oracle.wav` | Scrambled and Griffin-Lim recovered speech audio |
| `outputs/white_*dB.wav`, `outputs/babble_*dB.wav` | Generated noisy speech across SNR levels |

---

## Limitations

- Evaluated on a single 10-second speech recording with single noise excerpts per noise category.
- Random phase evaluations use a single fixed random seed (`seed = 0`) unless otherwise specified.
- STOI measures envelope correlation over ~26 ms blocks and is largely insensitive to sub-frame phase destruction.
