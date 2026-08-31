# 03 · Signal Processing Basics

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Signal Processing Toolbox behavior (`fft`, `filter`, `designfilt`) is
    hand-traced against documented MATLAB semantics and cross-checked
    against NumPy/SciPy equivalents where feasible, rather than run in
    MATLAB itself.

A **signal** is just a vector of samples taken over time. Everything in
this module — analyzing frequency content, removing noise, designing
filters — is standard MATLAB numeric work applied to that one idea, using
functions that have been production-stable for decades.

## Sampling and the sample rate

A continuous physical signal becomes a MATLAB vector by sampling it at a
fixed rate `Fs` (samples/second, Hz). The vector alone has no notion of
time — you must track `Fs` alongside it to interpret the data correctly:

```matlab
Fs = 1000;                     % 1000 samples per second
t  = (0:1/Fs:1-1/Fs)';         % 1 second of time stamps
f0 = 50;                        % a 50 Hz tone
x  = sin(2*pi*f0*t) + 0.5*randn(size(t));  % signal + noise
```

The **Nyquist theorem**: a sampling rate `Fs` can only faithfully
represent frequencies up to `Fs/2` (the *Nyquist frequency*). Sample a
120 Hz tone at `Fs = 100` and it does not simply disappear — it
**aliases**, reappearing disguised as a lower, wrong frequency
(`|120 - 100| = 20 Hz`). This is why every signal-processing pipeline
starts by confirming `Fs` is at least twice the highest frequency of
interest.

## Frequency-domain view: the FFT

`fft` converts a time-domain vector into its frequency-domain
representation — how much of the signal's energy sits at each frequency:

```matlab
N = length(x);
X = fft(x);
f = (0:N-1)*(Fs/N);            % frequency axis, 0 to Fs
mag = abs(X)/N;                 % magnitude, normalized by length

% Single-sided spectrum: real signals are symmetric about Fs/2,
% so the informative half is 1:N/2+1
halfN = floor(N/2)+1;
f_half   = f(1:halfN);
mag_half = mag(1:halfN);
mag_half(2:end-1) = 2*mag_half(2:end-1);  % account for folded energy

plot(f_half, mag_half);
xlabel('Frequency (Hz)'); ylabel('Magnitude');
```

For the `x` built above, `mag_half` shows a clear peak near `f = 50` (the
injected tone) sitting above a noise floor spread across all
frequencies — exactly what random Gaussian noise plus one sinusoid looks
like in the frequency domain. `abs()` discards phase and keeps magnitude;
`angle(X)` recovers phase when it matters (e.g., aligning signals).

`fftshift` re-centers the spectrum around 0 Hz instead of running 0→`Fs`,
convenient when a signal has meaningful negative-frequency content
(complex-valued signals, common in communications work) rather than the
real-valued case above.

## Filtering: keeping or removing frequency bands

A **filter** passes some frequencies and attenuates others. MATLAB's
`designfilt` builds a filter from a frequency-domain specification
without hand-deriving coefficients:

```matlab
d = designfilt('lowpassiir', ...
    'FilterOrder', 8, ...
    'HalfPowerFrequency', 100, ...   % Hz, the -3dB cutoff
    'SampleRate', Fs);

y = filter(d, x);      % apply the filter to the signal
```

`'lowpassiir'` keeps frequencies below the cutoff and attenuates above;
`'highpassiir'`, `'bandpassiir'`, `'bandstopiir'` cover the other common
shapes. `filtfilt(d, x)` applies the same filter forward and backward,
canceling the phase distortion a single-pass `filter` introduces —
preferred whenever a shifted-in-time output would corrupt downstream
analysis (e.g., detecting exact event timing) and only unsuitable for
real-time/streaming use, where the whole signal isn't available yet to
run backward over.

### FIR vs. IIR, briefly

| | FIR (`'lowpassfir'`) | IIR (`'lowpassiir'`) |
|---|---|---|
| Impulse response | Finite length | Theoretically infinite (feedback) |
| Phase | Can be made exactly linear | Generally nonlinear |
| Cost for a given sharpness | Higher order needed | Lower order suffices |
| Stability | Always stable | Can be unstable if poles land outside the unit circle |

IIR filters are cheaper (lower order for the same cutoff sharpness) but
distort phase and carry a stability condition; FIR filters cost more
compute for equivalent sharpness but are always stable and can preserve
linear phase — the right choice depends on whether phase fidelity or
computational cost dominates for the application.

## Moving average: the simplest filter of all

Before reaching for `designfilt`, a moving average is often enough to
smooth noisy data and is trivial to reason about:

```matlab
windowSize = 5;
b = ones(1, windowSize) / windowSize;
ySmooth = filter(b, 1, x);
```

This is literally `filter` with FIR coefficients `[1/5 1/5 1/5 1/5 1/5]`
and denominator `1` — every output sample is the average of the current
and previous four input samples. It attenuates high-frequency noise (a
crude low-pass filter) at the cost of a slight delay and softened edges,
which is why `designfilt` exists for anything needing a controlled,
specified cutoff rather than "whatever a 5-point average happens to do."

## `movmean`, `movmedian`, `smoothdata` — the convenience layer

For quick smoothing without building a formal filter object:

```matlab
ySmooth  = movmean(x, 5);        % same idea as the manual filter above
yDenoise = movmedian(x, 5);      % robust to spike outliers, mean isn't
ySmooth2 = smoothdata(x, 'gaussian', 11);  % weighted, smoother rolloff
```

`movmedian` matters specifically because a single large spike (a sensor
glitch) drags a moving *average* window's output toward the spike for its
whole width, while a moving *median* ignores an isolated outlier
entirely — the same mean-vs-median robustness trade-off from basic
statistics, applied along a sliding window.

## Spectrogram: frequency content that changes over time

`fft` describes one snapshot's overall frequency content; a signal whose
frequency content changes over time (a chirp, speech, a machine spinning
up) needs `spectrogram`, which computes the FFT over many short
overlapping windows and shows a frequency-vs-time image:

```matlab
spectrogram(x, 256, 200, 256, Fs, 'yaxis');
```

The three numeric arguments after `x` are the window length, overlap, and
FFT length — a common choice is roughly 75–80% overlap, trading time
resolution against frequency resolution the same way choosing a coarser
or finer FFT window always does.

## Summary

- A signal is a sampled vector plus its sample rate `Fs`; Nyquist
  (`Fs ≥ 2×`highest frequency`) is the first thing to check before
  trusting any analysis.
- `fft` moves a signal into the frequency domain; `abs(fft(x))` gives
  magnitude, and only the first half (`1:N/2+1`) of a real signal's
  spectrum is independent information.
- `designfilt` + `filter`/`filtfilt` build and apply low/high/band-pass
  filters from a frequency spec; `filtfilt` removes phase distortion at
  the cost of needing the whole signal up front.
- FIR filters cost more per unit of sharpness but are always stable and
  can have exactly linear phase; IIR filters are cheaper but can be
  unstable and generally distort phase.
- `movmean`/`movmedian`/`smoothdata` cover quick smoothing without a
  formal filter design; `movmedian` specifically resists outlier spikes
  that would drag a moving average off course.
- `spectrogram` extends `fft` to signals whose frequency content changes
  over time.
