# 06 · App Designer Basics

!!! note "Verification note"
    MATLAB App Designer was not available in the environment used to
    write this page. The `classdef`-based app structure, callback
    wiring, and component properties described below are documented,
    version-stable App Designer conventions, hand-traced against the
    MATLAB documentation rather than built or run in App Designer
    itself.

App Designer builds interactive GUI applications on top of the same
`classdef` mechanics from Module 01, generating a `.mlapp` file — a
zipped bundle containing a classdef, the UI layout, and resources.
Rather than write layout code by hand (though that's possible via
`uifigure` + programmatic component creation), App Designer's canvas
generates that code for you; this module focuses on the code structure
that either path produces, since that's what you can read, review, and
extend.

## Anatomy of an app

An App Designer app is a `handle` class (Module 01 — because UI state
must be shared and mutated across callbacks, not copied):

```matlab
classdef TemperatureConverterApp < matlab.apps.AppBase

    properties (Access = public)
        UIFigure          matlab.ui.Figure
        CelsiusField      matlab.ui.control.NumericEditField
        FahrenheitField   matlab.ui.control.NumericEditField
        ConvertButton     matlab.ui.control.Button
        CelsiusLabel      matlab.ui.control.Label
        FahrenheitLabel   matlab.ui.control.Label
    end

    methods (Access = private)

        function ConvertButtonPushed(app, event)
            c = app.CelsiusField.Value;
            f = c * 9/5 + 32;
            app.FahrenheitField.Value = f;
        end

    end

    methods (Access = private)

        function createComponents(app)
            app.UIFigure = uifigure('Name', 'Temperature Converter', 'Position', [100 100 300 200]);

            app.CelsiusLabel = uilabel(app.UIFigure, 'Text', 'Celsius:', 'Position', [30 130 60 22]);
            app.CelsiusField = uieditfield(app.UIFigure, 'numeric', 'Position', [100 130 100 22]);

            app.FahrenheitLabel = uilabel(app.UIFigure, 'Text', 'Fahrenheit:', 'Position', [30 90 60 22]);
            app.FahrenheitField = uieditfield(app.UIFigure, 'numeric', 'Position', [100 90 100 22]);
            app.FahrenheitField.Editable = 'off';

            app.ConvertButton = uibutton(app.UIFigure, 'push', ...
                'Text', 'Convert', ...
                'Position', [100 50 100 30], ...
                'ButtonPushedFcn', createCallbackFcn(app, @ConvertButtonPushed, true));
        end

    end

    methods (Access = public)

        function app = TemperatureConverterApp
            createComponents(app);
            registerApp(app, app.UIFigure);
        end

        function delete(app)
            delete(app.UIFigure);
        end

    end
end
```

Three parts matter:

1. **Properties** declare every UI component as a typed property so
   callbacks can reach `app.CelsiusField.Value` from anywhere in the
   class.
2. **`createComponents`** builds the figure and every widget once, at
   construction, wiring each control's callback to a method via
   `createCallbackFcn`.
3. **Callback methods** (`ConvertButtonPushed`) run whenever their
   event fires, receiving `app` (the handle object — so mutations
   persist) and `event` (details about what triggered the callback).

## `uifigure` vs. classic `figure`

App Designer apps run on `uifigure`, a separate figure type from the
plotting `figure` used in earlier modules. `uifigure` supports the
modern app-building components (`uieditfield`, `uidropdown`,
`uiswitch`, `uigauge`, `uitable`, `uitree`, `uiaxes` for plots inside
apps) but does *not* support some legacy plotting commands directly on
it — plotting inside an app goes through a `uiaxes` component:

```matlab
app.UIAxes = uiaxes(app.UIFigure, 'Position', [20 20 260 150]);
plot(app.UIAxes, x, y);        % target the uiaxes explicitly, not gca
```

## Common components

```matlab
uilabel(fig, 'Text', 'Status:');
uieditfield(fig, 'text');                 % text input
uieditfield(fig, 'numeric');              % numeric input, auto-validates
uibutton(fig, 'push', 'Text', 'Run');
uidropdown(fig, 'Items', {'Low','Medium','High'});
uicheckbox(fig, 'Text', 'Enable logging');
uislider(fig, 'Limits', [0 100]);
uiswitch(fig, 'rocker', 'Items', {'Off','On'});
uitable(fig, 'Data', magic(4));
uigauge(fig, 'Limits', [0 100]);
uilistbox(fig, 'Items', {'A','B','C'}, 'Multiselect', 'on');
```

Every component's callback follows the same naming convention:
`ValueChangedFcn` for controls that carry a value (dropdown, slider,
checkbox), `ButtonPushedFcn` for buttons, `SelectionChangedFcn` for
lists/tables.

## Wiring callbacks and sharing state

Callbacks read and write `app` properties directly, which is why apps
must be handle classes — a value class would give each callback its own
disconnected copy of the app, breaking state persistence between calls.

```matlab
properties (Access = private)
    HistoryLog = {}     % private state not exposed as a UI component
end

methods (Access = private)

    function ConvertButtonPushed(app, event)
        c = app.CelsiusField.Value;
        f = c * 9/5 + 32;
        app.FahrenheitField.Value = f;

        app.HistoryLog{end+1} = sprintf('%.1fC -> %.1fF', c, f);
        app.HistoryListBox.Items = app.HistoryLog;
    end

end
```

## Startup and cleanup

```matlab
methods (Access = private)

    function startupFcn(app, varargin)
        app.HistoryLog = {};
        app.StatusLabel.Text = 'Ready';
    end

end
```

`startupFcn`, if present, is called automatically right after
`createComponents` during construction — the natural place to
initialize data, load a config file, or set default control values
before the user interacts with anything.

`delete(app)` is the destructor: closing the figure calls it, and it
should tear down anything that outlives the figure itself — open files,
timers, or connections.

## Timers for periodic UI updates

```matlab
properties (Access = private)
    UpdateTimer
end

methods (Access = private)

    function startupFcn(app)
        app.UpdateTimer = timer('ExecutionMode', 'fixedRate', ...
                                 'Period', 1, ...
                                 'TimerFcn', @(~,~) app.refreshClock());
        start(app.UpdateTimer);
    end

    function refreshClock(app)
        app.ClockLabel.Text = datestr(now, 'HH:MM:SS');
    end

    function delete(app)
        if isvalid(app.UpdateTimer)
            stop(app.UpdateTimer);
            delete(app.UpdateTimer);
        end
        delete(app.UIFigure);
    end

end
```

Always stop and delete timers in `delete(app)` — an app closed with a
running timer leaves an orphaned timer object firing callbacks against a
deleted figure, which errors repeatedly in the background.

## Input validation

```matlab
function CelsiusFieldValueChanged(app, event)
    c = app.CelsiusField.Value;
    if c < -273.15
        uialert(app.UIFigure, 'Temperature below absolute zero is not physical.', 'Invalid Input');
        app.CelsiusField.Value = -273.15;
        return;
    end
    app.ConvertButtonPushed(event);
end
```

`uialert` shows a modal dialog without halting the whole app the way
`error()` would — appropriate for user-facing validation feedback inside
an interactive tool, where you want the user to fix input and continue,
not crash the session.

## Practice

1. Extend the temperature converter with a `uidropdown` to choose the
   conversion direction (C→F or F→C) and update the single active edit
   field's `Editable` property accordingly.
2. Add a `uitable` that logs every conversion performed (input, output,
   timestamp) and a "Clear History" button that resets it.
3. Add a `startupFcn` that disables the Convert button until the
   Celsius field has a non-empty value, using its `ValueChangedFcn` to
   re-enable it.
4. Explain, in your own words, why an App Designer app's main class must
   inherit handle semantics rather than being a plain value `classdef`,
   referencing what breaks (concretely, in terms of what a callback
   would and wouldn't see) if it weren't.
