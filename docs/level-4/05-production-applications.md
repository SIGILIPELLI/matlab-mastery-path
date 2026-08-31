# 05 · Building Production MATLAB Applications

!!! note "Verification note"
    MATLAB Compiler and MATLAB Production Server were not available in
    the environment used to write this page. Packaging, deployment, and
    licensing behaviors below are documented, version-stable MathWorks
    product features, hand-traced against the MathWorks documentation
    rather than exercised against real installations.

Everything up to this point runs inside MATLAB, with MATLAB installed
and a valid license. Moving a MATLAB application into production —
handed to end users without MATLAB, or serving requests from other
systems — requires packaging and deployment tools this module covers:
MATLAB Compiler, MATLAB Production Server, and the surrounding
operational concerns (configuration, logging, versioning) that turn a
working script into a maintainable application.

## Standalone applications: MATLAB Compiler

```matlab
mcc -m myApp.m -o MyApplication
```

`mcc` (or the `Application Compiler` app, its GUI front-end) packages a
MATLAB program plus the MATLAB Runtime dependencies into a standalone
executable. The **MATLAB Runtime** (a free, separately installable
redistributable, not the full MATLAB IDE) is what actually runs the
compiled application on an end user's machine — end users need the
Runtime installed (a one-time step, often bundled into the app's
own installer), but not a MATLAB license.

```matlab
mcc -m myApp.m -o MyApplication -a supportingData.mat -a helperFunctions/
```

`-a` bundles additional files (data files, folders of helper functions)
into the compiled package so the application is self-contained rather
than depending on files existing at particular paths on the end user's
machine.

### What compiles and what doesn't

Compiled standalone applications have similar restrictions to MATLAB
Coder (Module 02) around dynamic behavior, though generally more
permissive since they still run through the MATLAB Runtime interpreter
rather than becoming native machine code — `eval`, dynamic field access,
and most full MATLAB semantics work in a compiled app, unlike in
Coder-generated C. The key restriction is licensing: a compiled app can
only use toolbox functions the *building* machine had licenses for, and
the generated app itself needs no further license to run (that's the
whole point), but toolbox-specific functionality baked in at compile
time can't later be extended without a license on the *build* machine.

## Web apps: MATLAB Web App Server / Compiler for web deployment

```matlab
mcc -W webapp:MyWebApp -T link:webapp myApp.m
```

Compiling for the web target produces an app deployable to MATLAB Web
App Server, accessible through a browser without any client-side MATLAB
or Runtime installation at all — appropriate for internal tools where
many users need occasional access to a MATLAB-built analysis tool
without installing anything locally.

## Production Server: serving MATLAB functions as an API

MATLAB Production Server exposes MATLAB functions as callable services
(REST/HTTP or a client library) that other applications integrate
against — the pattern for embedding a MATLAB-built algorithm (say, a
trained ML model or signal-processing pipeline from Level 3 Module 10)
into a larger non-MATLAB production system (a web backend, a mobile
app's server, another company's pipeline).

```matlab
function result = scoreApplicant(features)
    model = loadCompiledData('creditModel.mat');
    result = predict(model, features);
end
```

Deployed via Production Server, this function becomes callable over
HTTP by any client that can make a REST request — the calling system
(often written in Java, Python, or a web framework) never needs MATLAB
installed, sees only a JSON request/response contract.

```matlab
% .m file used to define which functions are exposed as a deployable archive
buildfile = compiler.build.ProductionServerArchive('scoreApplicant.m', ...
    'ArchiveName', 'CreditScoringService');
compiler.build.productionServerArchive(buildfile);
```

`loadCompiledData` (rather than `load`) is the Production
Server-specific pattern for loading data bundled into the deployed
archive — it caches the loaded data across requests so repeated calls
don't reload the model file from disk on every single request, which
matters for request latency in a production service handling many
calls.

## Configuration management

Production code shouldn't hardcode environment-specific values (file
paths, API endpoints, credentials) the way an exploratory script might:

```matlab
% BAD: hardcoded, breaks when deployed to a different environment
dataPath = '/Users/dev/project/data/';
```

```matlab
% BETTER: externalized configuration
function config = loadConfig()
    configFile = fullfile(getenv('APP_CONFIG_DIR'), 'config.json');
    config = jsondecode(fileread(configFile));
end

config = loadConfig();
data = readtable(fullfile(config.dataPath, 'input.csv'));
```

Reading configuration from an environment variable-pointed JSON/YAML
file means the same compiled application runs correctly in development,
staging, and production environments without recompiling — only the
external config file differs per environment.

## Logging for production diagnostics

```matlab
function logMessage(level, msg, varargin)
    ts = datestr(now, 'yyyy-mm-dd HH:MM:SS');
    formatted = sprintf(msg, varargin{:});
    fprintf('[%s] [%s] %s\n', ts, level, formatted);
    % in production, redirect this to a persistent log file rather than stdout:
    % fid = fopen(config.logPath, 'a'); fprintf(fid, ...); fclose(fid);
end

logMessage('INFO', 'Processing request for user %d', userId);
logMessage('ERROR', 'Failed to load model: %s', err.message);
```

A compiled standalone app or a Production Server function has no
interactive Command Window for a developer to watch — structured,
leveled logging to a file (or a logging service) is the only visibility
into what a deployed application is actually doing once it's out of a
developer's hands, especially for diagnosing a failure reported after
the fact.

## Error handling for unattended execution

```matlab
function result = safeProcessRequest(input)
    try
        validateInput(input);
        result = coreProcessing(input);
    catch ME
        logMessage('ERROR', 'Request failed: %s\nStack: %s', ...
            ME.message, getReport(ME, 'basic'));
        result = struct('success', false, 'error', ME.message);
    end
end
```

An interactive script can let an error propagate to the Command Window
for a human to read and fix. A deployed service must catch errors,
log full diagnostic detail (`getReport` gives a full stack trace),
and return a well-formed error response — an uncaught exception in a
production service typically means a crashed request or a hung worker
process, not a helpful message to a developer watching.

## Versioning compiled applications

```matlab
% embed version info so deployed instances are identifiable
APP_VERSION = '2.3.1';
logMessage('INFO', 'Starting MyApplication v%s', APP_VERSION);
```

Without an embedded version identifier, diagnosing "which build is
actually running in production" after multiple deployments becomes
guesswork — a minimal but essential production practice, paired with
the CI/CD pipeline discipline covered in Module 08 (build artifacts
tagged with version and commit hash).

## Choosing a deployment target

| Situation | Tool |
|---|---|
| Give an end user a runnable app, no MATLAB needed | MATLAB Compiler → standalone `.exe` |
| Browser-based internal tool, no client install | MATLAB Compiler → web app target / Web App Server |
| Expose a MATLAB algorithm as an API for other systems | MATLAB Production Server |
| Speed-critical numeric core embedded in a non-MATLAB app | MATLAB Coder (Module 02) instead — compiles to native C |

## Practice

1. Explain the difference between what a MATLAB Compiler standalone app
   needs on the end-user's machine versus what MATLAB Coder-generated C
   code needs, and why that distinction affects which one you'd choose
   to embed a real-time control algorithm into a microcontroller versus
   distributing a desktop analysis tool.
2. Rewrite a script that hardcodes a data folder path and a threshold
   constant to instead read both from an externalized JSON config file,
   and explain what would break if the config file is missing versus
   what should happen (a clear startup error vs. a silent wrong
   default).
3. Design the logging calls (level, message, what to include) you'd add
   to the credit-scoring `scoreApplicant` example so that a failure in
   production, investigated days later from logs alone, gives enough
   information to reproduce and diagnose it.
4. Explain why an uncaught MATLAB error inside a Production Server
   function is a more serious operational problem than the same error
   occurring in an interactive script, and what the `safeProcessRequest`
   pattern does to prevent it from taking down the whole service.
