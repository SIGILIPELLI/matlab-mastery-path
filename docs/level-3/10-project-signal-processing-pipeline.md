# 10 · Project — Signal Processing Pipeline

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. All FFT, filter design, and signal generation code below was
    hand-traced against documented Signal Processing Toolbox and core
    MATLAB semantics, and cross-checked numerically (where the
    computation is expressible without MATLAB-only toolbox internals)
    against equivalent NumPy/SciPy logic, rather than executed in
    MATLAB itself.

This capstone pulls together OOP (Module 01), numerical methods (Module
08), and testing (Module 09) into one project: a reusable pipeline that
generates a noisy signal, filters it, analyzes its frequency content,
and reports detected features — the kind of end-to-end structure a real
signal-processing task needs, packaged as a class rather than a loose
script.

## Problem statement

Build a `SignalPipeline` class that:

1. Generates (or accepts) a signal composed of known sine components
   plus noise.
2. Applies a configurable filter (lowpass/highpass/bandpass) to remove
   unwanted frequency content.
3. Computes the FFT to identify dominant frequencies.
4. Reports the detected peak frequencies and their magnitudes.
5. Is covered by a unit test suite validating each stage independently.

## Step 1 — Signal generation

```matlab
classdef SignalPipeline < handle

    properties
        Fs              % sampling frequency (Hz)
        Duration        % signal duration (s)
        RawSignal
        FilteredSignal
        Time
    end

    methods
        function obj = SignalPipeline(fs, duration)
            obj.Fs = fs;
            obj.Duration = duration;
            obj.Time = (0:1/fs:duration - 1/fs)';   % column vector of sample times
        end

        function generateSignal(obj, freqsHz, amplitudes, noiseStd)
            if nargin < 4
                noiseStd = 0;
            end
            obj.RawSignal = zeros(size(obj.Time));
            for k = 1:numel(freqsHz)
                obj.RawSignal = obj.RawSignal + ...
                    amplitudes(k) * sin(2*pi*freqsHz(k)*obj.Time);
            end
            if noiseStd > 0
                obj.RawSignal = obj.RawSignal + noiseStd * randn(size(obj.Time));
            end
        end
    end
end
```

Hand-trace: with `fs = 1000`, `duration = 1`, `obj.Time` has 1000
samples spanning `[0, 0.999]` seconds — the Nyquist frequency is `fs/2 =
500 Hz`, so any generated component above 500 Hz would alias and must be
avoided in test signals.

```matlab
p = SignalPipeline(1000, 1);
p.generateSignal([50, 120], [1.0, 0.5], 0.2);   % 50 Hz + 120 Hz tones, Gaussian noise std 0.2
```

## Step 2 — Filtering

```matlab
methods
    function applyFilter(obj, type, cutoffHz, order)
        if nargin < 4
            order = 4;
        end
        nyquist = obj.Fs / 2;
        switch type
            case 'lowpass'
                Wn = cutoffHz / nyquist;
                [b, a] = butter(order, Wn, 'low');
            case 'highpass'
                Wn = cutoffHz / nyquist;
                [b, a] = butter(order, Wn, 'high');
            case 'bandpass'
                Wn = cutoffHz / nyquist;   % cutoffHz = [low, high]
                [b, a] = butter(order, Wn, 'bandpass');
            otherwise
                error('SignalPipeline:invalidFilterType', ...
                      'Unknown filter type: %s', type);
        end
        obj.FilteredSignal = filtfilt(b, a, obj.RawSignal);
    end
end
```

`butter` designs a Butterworth filter — maximally flat passband, no
ripple — as a good general-purpose default. Cutoff frequencies are
normalized by the Nyquist frequency (`Wn` in `[0, 1]`, where `1`
corresponds to `fs/2`); passing raw Hz values directly to `butter`
without this normalization is a common bug that silently produces a
badly wrong filter.

`filtfilt` (rather than `filter`) applies the filter forward and then
backward, canceling the phase distortion a single-pass IIR filter
introduces — critical when the *timing* of filtered features matters,
at the cost of needing the entire signal in memory (not suitable for
real-time streaming, where causal `filter` is required instead).

```matlab
p.applyFilter('lowpass', 80, 4);   % keep the 50 Hz tone, attenuate the 120 Hz tone
```

## Step 3 — Frequency analysis via FFT

```matlab
methods
    function [freqs, magnitudes] = computeSpectrum(obj, useFiltered)
        if nargin < 2
            useFiltered = true;
        end
        signal = obj.FilteredSignal;
        if ~useFiltered || isempty(signal)
            signal = obj.RawSignal;
        end

        N = length(signal);
        Y = fft(signal);
        Y = Y(1:floor(N/2)+1);              % keep only the non-redundant half (real input)
        magnitudes = abs(Y) / N;
        magnitudes(2:end-1) = 2 * magnitudes(2:end-1);  % account for the folded negative frequencies

        freqs = (0:floor(N/2))' * (obj.Fs / N);
    end
end
```

Hand-trace of the scaling: `fft` on a real signal produces a spectrum
symmetric about the Nyquist frequency, so all the energy at, say, +50 Hz
is split between the +50 Hz and -50 Hz bins in the full spectrum. Taking
only the first half and doubling every bin except DC (`Y(1)`) and (for
even N) the Nyquist bin (`Y(end)`) recovers the correct single-sided
amplitude — this is why `magnitudes(2:end-1)` (excluding both ends) is
doubled, not the whole vector.

For the example signal (`fs=1000`, `N=1000`), frequency resolution is
`fs/N = 1 Hz` per bin — `freqs` runs `0, 1, 2, ..., 500` Hz, and a pure
50 Hz sine of amplitude 1.0 should produce a peak magnitude near 1.0 at
`freqs(51)` (index 51 corresponds to 50 Hz, since `freqs(1)` is 0 Hz).

## Step 4 — Peak detection and reporting

```matlab
methods
    function report = detectPeaks(obj, minProminence)
        if nargin < 2
            minProminence = 0.1;
        end
        [freqs, mags] = obj.computeSpectrum();
        [peakMags, peakLocs] = findpeaks(mags, 'MinPeakProminence', minProminence);
        peakFreqs = freqs(peakLocs);

        [sortedMags, order] = sort(peakMags, 'descend');
        sortedFreqs = peakFreqs(order);

        report = table(sortedFreqs, sortedMags, ...
            'VariableNames', {'FrequencyHz', 'Magnitude'});
    end
end
```

`findpeaks` with `'MinPeakProminence'` filters out noise-floor bumps
that aren't real spectral peaks — prominence measures how much a peak
stands out from its surrounding baseline, a more robust criterion than
a raw magnitude threshold when the noise floor itself varies across the
spectrum.

## Step 5 — Putting it together

```matlab
p = SignalPipeline(1000, 1);
p.generateSignal([50, 120], [1.0, 0.5], 0.2);
p.applyFilter('lowpass', 80, 4);
report = p.detectPeaks(0.1);
disp(report);
% Expect: one dominant peak near 50 Hz, magnitude near 1.0;
% the 120 Hz tone should be strongly attenuated by the 80 Hz lowpass
% and either absent or far below the 50 Hz peak's magnitude.

figure;
subplot(2,1,1);
plot(p.Time, p.RawSignal, p.Time, p.FilteredSignal);
legend('Raw', 'Filtered');
xlabel('Time (s)'); ylabel('Amplitude');

subplot(2,1,2);
[freqs, mags] = p.computeSpectrum();
plot(freqs, mags);
xlabel('Frequency (Hz)'); ylabel('Magnitude');
title('Filtered Signal Spectrum');
```

## Step 6 — Unit tests for the pipeline

```matlab
classdef SignalPipelineTest < matlab.unittest.TestCase

    methods (Test)

        function testGenerateSignalLength(testCase)
            p = SignalPipeline(1000, 1);
            p.generateSignal(50, 1, 0);
            testCase.verifyEqual(length(p.RawSignal), 1000);
        end

        function testPureToneDetectedAtCorrectFrequency(testCase)
            p = SignalPipeline(1000, 1);
            p.generateSignal(50, 1, 0);       % no noise, no filtering — clean test
            [freqs, mags] = p.computeSpectrum(false);
            [~, idx] = max(mags);
            testCase.verifyEqual(freqs(idx), 50, 'AbsTol', 1);
        end

        function testLowpassAttenuatesHighFrequency(testCase)
            p = SignalPipeline(1000, 1);
            p.generateSignal([50, 200], [1, 1], 0);
            p.applyFilter('lowpass', 80, 4);
            [freqs, mags] = p.computeSpectrum(true);
            mag50 = mags(freqs == 50);
            mag200 = mags(freqs == 200);
            testCase.verifyLessThan(mag200, mag50 * 0.1);  % strongly attenuated relative to passband
        end

        function testInvalidFilterTypeThrows(testCase)
            p = SignalPipeline(1000, 1);
            p.generateSignal(50, 1, 0);
            testCase.verifyError(@() p.applyFilter('notAFilter', 80), ...
                'SignalPipeline:invalidFilterType');
        end

        function testDetectPeaksReturnsTable(testCase)
            p = SignalPipeline(1000, 1);
            p.generateSignal([50, 150], [1, 0.8], 0);
            report = p.detectPeaks(0.1);
            testCase.verifyClass(report, 'table');
            testCase.verifyGreaterThanOrEqual(height(report), 2);
        end

    end
end
```

`testLowpassAttenuatesHighFrequency` is the most important test here —
it verifies the *functional intent* of the filter stage (unwanted
frequencies suppressed relative to the passband), not just that
`applyFilter` runs without erroring.

## Extending the project

- Add a `spectrogram`-based method for time-varying frequency content
  (useful once a signal's frequency composition changes over its
  duration, which a single FFT over the whole signal can't show).
- Add a `SNR` property/method computing signal-to-noise ratio before and
  after filtering, to quantify how much the filter helped.
- Parameterize the test suite (Module 09) over several `(freq, cutoff)`
  combinations to check the filter's behavior systematically rather
  than at one hand-picked operating point.
- Package the class as part of a small toolbox (Level 4 covers
  packaging and distribution) so it's reusable across projects without
  copying the file.

## Practice

1. Implement the `spectrogram`-based extension described above and
   write a test that generates a signal whose frequency changes halfway
   through (e.g. 50 Hz for the first half-second, 150 Hz for the
   second) and verifies the spectrogram shows the transition at
   approximately the right time index.
2. Add a bandpass example isolating a middle frequency out of three
   generated tones, and a test asserting the two flanking tones are both
   attenuated relative to the passband tone.
3. Investigate (by reasoning through `filtfilt`'s forward-backward
   mechanism) why `filtfilt` effectively doubles the filter's order in
   terms of attenuation steepness compared to a single `filter` pass
   with the same `b`, `a` coefficients.
4. Extend `detectPeaks` to also report each peak's estimated bandwidth
   (e.g. via `findpeaks`'s `'WidthReference'` output), and discuss why
   frequency resolution (`fs/N`) puts a floor on how precisely two close
   peaks can be distinguished regardless of filter design.
