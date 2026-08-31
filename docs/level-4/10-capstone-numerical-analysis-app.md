# 10 · Capstone — Full Numerical Analysis Application

!!! note "Verification note"
    MATLAB, App Designer, and MATLAB Compiler were not available in the
    environment used to write this page. Every code fragment below was
    hand-traced against documented MATLAB semantics from earlier
    modules in this course, rather than built or run in MATLAB itself.

This capstone integrates the entire course into one project: a
numerical analysis desktop application that lets a user load data,
choose an analysis (root-finding, curve fitting, ODE simulation, or the
Level 3 signal-processing pipeline), configure it through a GUI,
visualize results, and export a report — packaged as a testable,
deployable application rather than a script.

## Architecture

```
NumericalAnalysisApp (App Designer classdef, handle) — UI layer
        |
        v
AnalysisEngine (handle class) — coordinates the selected analysis
        |
        +-- RootFinder        (wraps fzero/roots, Level 3 Module 08)
        +-- CurveFitter       (wraps polyfit/fit, Level 2)
        +-- OdeSimulator      (wraps ode45/ode15s, Level 3 Module 08)
        +-- SignalPipeline    (Level 3 Module 10, reused unchanged)
        |
        v
ReportGenerator — exports results as a PDF/HTML summary
```

Separating the UI (`NumericalAnalysisApp`) from the computation
(`AnalysisEngine` and its sub-analyzers) means the analysis logic is
independently unit-testable (Level 3 Module 09) without needing App
Designer or a display at all — exactly the same separation of concerns
argued for throughout the course.

## The analysis engine

```matlab
classdef AnalysisEngine < handle

    properties
        LastResult
        LastAnalysisType
    end

    methods
        function result = runRootFinding(obj, funcStr, guess)
            f = str2func(funcStr);
            root = fzero(f, guess);
            result = struct('type', 'root', 'root', root, 'functionValue', f(root));
            obj.LastResult = result;
            obj.LastAnalysisType = 'RootFinding';
        end

        function result = runCurveFit(obj, x, y, degree)
            coeffs = polyfit(x, y, degree);
            yFit = polyval(coeffs, x);
            residuals = y - yFit;
            ssRes = sum(residuals.^2);
            ssTot = sum((y - mean(y)).^2);
            rSquared = 1 - ssRes/ssTot;
            result = struct('type', 'curveFit', 'coefficients', coeffs, ...
                             'rSquared', rSquared, 'fitted', yFit);
            obj.LastResult = result;
            obj.LastAnalysisType = 'CurveFit';
        end

        function result = runOdeSimulation(obj, odeFuncStr, tspan, y0)
            f = str2func(odeFuncStr);
            [t, y] = ode45(f, tspan, y0);
            result = struct('type', 'ode', 'time', t, 'states', y);
            obj.LastResult = result;
            obj.LastAnalysisType = 'OdeSimulation';
        end

        function result = runSignalAnalysis(obj, fs, duration, freqs, amps, noiseStd, filterCutoff)
            pipeline = SignalPipeline(fs, duration);
            pipeline.generateSignal(freqs, amps, noiseStd);
            pipeline.applyFilter('lowpass', filterCutoff, 4);
            peaks = pipeline.detectPeaks(0.1);
            result = struct('type', 'signal', 'pipeline', pipeline, 'peaks', peaks);
            obj.LastResult = result;
            obj.LastAnalysisType = 'SignalAnalysis';
        end
    end
end
```

`str2func` (Module 04's performance-and-safety-preferred alternative to
`eval`) converts a user-typed function string like `'@(x) x^2 - 4'` into
a callable handle — this is how the app lets a user type an arbitrary
function expression into a text field without resorting to `eval` on
untrusted user input.

## The App Designer front-end (structural sketch)

```matlab
classdef NumericalAnalysisApp < matlab.apps.AppBase

    properties (Access = public)
        UIFigure
        AnalysisTypeDropdown
        InputPanel
        ResultsTextArea
        ResultsAxes
        RunButton
        ExportButton
    end

    properties (Access = private)
        Engine
    end

    methods (Access = private)

        function startupFcn(app)
            app.Engine = AnalysisEngine();
        end

        function AnalysisTypeDropdownValueChanged(app, event)
            app.rebuildInputPanel(app.AnalysisTypeDropdown.Value);
        end

        function RunButtonPushed(app, event)
            try
                switch app.AnalysisTypeDropdown.Value
                    case 'Root Finding'
                        result = app.Engine.runRootFinding(app.getFuncInput(), app.getGuessInput());
                    case 'Curve Fit'
                        [x, y] = app.getXYData();
                        result = app.Engine.runCurveFit(x, y, app.getDegreeInput());
                    case 'ODE Simulation'
                        result = app.Engine.runOdeSimulation(app.getOdeFuncInput(), ...
                            app.getTspanInput(), app.getY0Input());
                    case 'Signal Analysis'
                        result = app.Engine.runSignalAnalysis(app.getFsInput(), ...
                            app.getDurationInput(), app.getFreqsInput(), ...
                            app.getAmpsInput(), app.getNoiseInput(), app.getCutoffInput());
                end
                app.displayResult(result);
            catch ME
                uialert(app.UIFigure, ME.message, 'Analysis Failed');
            end
        end

        function ExportButtonPushed(app, event)
            [file, path] = uiputfile('*.html', 'Save Report As');
            if isequal(file, 0)
                return;   % user cancelled — not an error
            end
            ReportGenerator.export(app.Engine.LastResult, app.Engine.LastAnalysisType, ...
                fullfile(path, file));
        end

    end
end
```

The `try`/`catch` around `RunButtonPushed`, with `uialert` reporting the
failure, follows the Module 05 (this level) production discipline of
never letting an unattended-facing error crash the whole application —
a malformed function string or empty data field should produce a
friendly error dialog, not a stack trace dumped to a Command Window the
end user of a compiled standalone app won't even see.

## Displaying results

```matlab
methods (Access = private)
    function displayResult(app, result)
        switch result.type
            case 'root'
                cla(app.ResultsAxes);
                text(app.ResultsAxes, 0.1, 0.5, ...
                    sprintf('Root: %.6f\nf(root): %.2e', result.root, result.functionValue));
                app.ResultsTextArea.Value = sprintf('Root found at x = %.6f', result.root);

            case 'curveFit'
                plot(app.ResultsAxes, 1:numel(result.fitted), result.fitted, 'r-');
                app.ResultsTextArea.Value = sprintf('R^2 = %.4f\nCoefficients: %s', ...
                    result.rSquared, mat2str(result.coefficients, 4));

            case 'ode'
                plot(app.ResultsAxes, result.time, result.states);
                app.ResultsTextArea.Value = sprintf('Simulated %d time points.', numel(result.time));

            case 'signal'
                plot(app.ResultsAxes, result.pipeline.Time, result.pipeline.FilteredSignal);
                app.ResultsTextArea.Value = sprintf('Detected %d peaks. Top: %.1f Hz', ...
                    height(result.peaks), result.peaks.FrequencyHz(1));
        end
    end
end
```

## Report generation

```matlab
classdef ReportGenerator

    methods (Static)
        function export(result, analysisType, outputPath)
            html = sprintf('<html><body><h1>%s Report</h1>', analysisType);
            html = [html, sprintf('<p>Generated: %s</p>', datestr(now))];

            fields = fieldnames(result);
            for i = 1:numel(fields)
                f = fields{i};
                if ~isstruct(result.(f)) && ~isa(result.(f), 'SignalPipeline')
                    html = [html, sprintf('<p><b>%s:</b> %s</p>', f, mat2str(result.(f), 4))];
                end
            end
            html = [html, '</body></html>'];

            fid = fopen(outputPath, 'w');
            fprintf(fid, '%s', html);
            fclose(fid);
        end
    end

end
```

`ReportGenerator` as a `Static`-methods-only class (no instance state
needed) is a simple utility grouping — appropriate here since report
generation is a pure function of its inputs with no state to maintain
across calls.

## Testing the engine independently of the UI

```matlab
classdef AnalysisEngineTest < matlab.unittest.TestCase

    methods (Test)

        function testRootFindingKnownRoot(testCase)
            engine = AnalysisEngine();
            result = engine.runRootFinding('@(x) x^2 - 4', 1);
            testCase.verifyEqual(result.root, 2, 'AbsTol', 1e-6);
        end

        function testCurveFitRSquaredNearOneForLinearData(testCase)
            engine = AnalysisEngine();
            x = 1:10;
            y = 2*x + 3;   % perfectly linear, noise-free
            result = engine.runCurveFit(x, y, 1);
            testCase.verifyEqual(result.rSquared, 1, 'AbsTol', 1e-10);
        end

        function testOdeSimulationMatchesAnalytical(testCase)
            engine = AnalysisEngine();
            result = engine.runOdeSimulation('@(t,y) -2*y', [0 5], 1);
            analytical = exp(-2*result.time);
            testCase.verifyEqual(result.states, analytical, 'AbsTol', 1e-4);
        end

        function testSignalAnalysisFindsExpectedPeak(testCase)
            engine = AnalysisEngine();
            result = engine.runSignalAnalysis(1000, 1, 50, 1, 0, 200);
            testCase.verifyEqual(result.peaks.FrequencyHz(1), 50, 'AbsTol', 1);
        end

    end
end
```

Because `AnalysisEngine` has no dependency on `NumericalAnalysisApp` or
any UI component, this entire suite runs in CI (Module 08) headlessly —
the UI itself would need App Designer's interactive testing tools
(outside this course's scope) to test directly, but keeping the engine
UI-independent means the vast majority of the application's actual
logic is tested this way regardless.

## Deployment

```matlab
mcc -m NumericalAnalysisApp.mlapp -o NumericalAnalysisApp -a src/
```

Following Module 05, the finished app compiles to a standalone
executable bundling `AnalysisEngine`, `SignalPipeline`, and
`ReportGenerator` (via `-a src/`) so end users run it without a MATLAB
license — the natural endpoint of everything this course built toward.

## What this capstone demonstrates end-to-end

- **Level 1-2**: the underlying numeric operations (polynomial fitting,
  matrix work) inside each analyzer.
- **Level 3**: OOP structure (`handle` classes throughout), the
  `SignalPipeline` reused verbatim, and the `matlab.unittest`-based test
  suite.
- **Level 4**: App Designer for the UI, `str2func` and `try`/`catch`
  discipline for production robustness (Modules 04-05), CI-testable
  architecture (Module 08), and `mcc` compilation for deployment
  (Module 05).

## Practice

1. Add a fifth analysis type (numerical integration via `integral`,
   Level 3 Module 08) following the same `AnalysisEngine` method +
   `displayResult` case + unit test pattern as the four already present.
2. Extend `ReportGenerator` to embed a plot image (via `saveas` on a
   hidden figure, then embedding it as a base64 `<img>` tag) rather than
   just numeric fields, and describe the tradeoff versus keeping the
   report purely textual.
3. Write a test verifying that `RunButtonPushed`'s error handling (not
   directly testable without App Designer, so describe conceptually)
   would correctly catch a malformed function string like `'x^2 - 4'`
   missing its `@(x)` prefix, and what error message the user should
   see instead of a raw MATLAB parse error.
4. Reflecting on the whole course, identify which single module's
   technique you'd reach for first if asked to harden this capstone
   application for production use with real users, and justify the
   choice.
