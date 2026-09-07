# 08 · CI/CD for MATLAB Projects

!!! note "Verification note"
    MATLAB, GitHub Actions with MATLAB support, and Jenkins were not
    available in the environment used to write this page. Pipeline
    configuration and MATLAB CI/CD tooling described below are
    documented, version-stable features of MathWorks' CI/CD
    integrations, hand-traced against the MathWorks documentation
    rather than run against a real pipeline.

Level 3 Module 09 built a unit test suite. This module wires that suite
(and code-quality checks) into automated pipelines that run on every
commit — catching regressions before they reach production, the way
CI/CD does for any other language.

## Why CI/CD matters for MATLAB specifically

MATLAB projects are sometimes treated as "just scripts," skipping the
CI discipline standard in software engineering. But the same failure
modes apply: a change to a shared helper function silently breaking a
downstream analysis, a refactor introducing a sign error only caught
weeks later. Automated testing on every push catches this immediately,
before it reaches a report, a deployed model, or a production service
(Module 05).

## Running MATLAB tests from the command line

```bash
matlab -batch "results = runtests('tests/', 'IncludeSubfolders', true); assertSuccess(results);"
```

`-batch` runs MATLAB non-interactively, executing the given command and
exiting with a status code — `0` on success, non-zero if `assertSuccess`
throws because any test failed. This exit code is exactly what a CI
system uses to mark a build passed or failed.

```bash
matlab -batch "runProjectTests"
```

Wrapping the test invocation in a project-specific script
(`runProjectTests.m`) keeps the CI configuration itself minimal and lets
the actual test logic (which folders, which plugins, coverage
reporting) live in version-controlled MATLAB code rather than pipeline
YAML.

## GitHub Actions

```yaml
# .github/workflows/matlab-ci.yml
name: MATLAB CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up MATLAB
        uses: matlab-actions/setup-matlab@v2

      - name: Run tests
        uses: matlab-actions/run-tests@v2
        with:
          source-folder: src
          test-results-junit: test-results/results.xml
          code-coverage-cobertura: coverage/coverage.xml

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: MATLAB Tests
          path: test-results/results.xml
          reporter: java-junit
```

MathWorks publishes official GitHub Actions (`matlab-actions/setup-matlab`,
`matlab-actions/run-tests`, `matlab-actions/run-command`) that handle
installing MATLAB on the runner and invoking the test suite, producing
standard formats (JUnit XML, Cobertura coverage XML) that other
CI-ecosystem tools (test reporters, coverage dashboards) already know
how to consume — avoiding the need to hand-roll MATLAB-to-CI glue code.

```yaml
      - name: Run static code analysis
        uses: matlab-actions/run-command@v2
        with:
          command: codeIssues = checkcode(genpath('src'), '-cyc'); disp(codeIssues);
```

## Jenkins pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh '''
                    matlab -batch "results = runtests('tests/', 'IncludeSubfolders', true); \
                    table(results), assertSuccess(results);"
                '''
            }
        }
        stage('Build Package') {
            steps {
                sh 'matlab -batch "mcc -m src/myApp.m -o build/MyApplication"'
            }
        }
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build/**', fingerprint: true
            }
        }
    }
    post {
        always {
            junit 'test-results/*.xml'
        }
    }
}
```

The same pattern (checkout, test, build, archive) applies regardless of
CI system — MATLAB's `-batch` command-line invocation is what makes it
composable with any CI tool that can run a shell command, not just
MATLAB-specific integrations.

## Static analysis in the pipeline

```matlab
issues = checkcode('src/myFunction.m', '-struct');
for i = 1:numel(issues)
    fprintf('Line %d: %s (%s)\n', issues(i).line, issues(i).message, issues(i).id);
end
```

`checkcode` (the command-line interface to the MATLAB Code Analyzer —
the same linter that annotates the Editor with warnings) can be run
headlessly in CI, failing the build on serious issues (undefined
variables, unreachable code) even before tests run — catching
structural problems earlier and cheaper than a failing test would.

```matlab
% Fail CI on any Code Analyzer warning above a severity threshold
issues = checkcode('src/', '-struct');
severeIssues = issues(strcmp({issues.id}, 'CABE') | strcmp({issues.id}, 'SUSP'));
if ~isempty(severeIssues)
    error('CI:codeAnalysisFailed', 'Found %d severe code issues.', numel(severeIssues));
end
```

## Test coverage gates

```matlab
import matlab.unittest.plugins.CodeCoveragePlugin
import matlab.unittest.plugins.codecoverage.CoverageResult

runner = matlab.unittest.TestRunner.withTextOutput;
coverageResult = CoverageResult;
runner.addPlugin(CodeCoveragePlugin.forFolder('src', 'Producing', coverageResult));

results = runner.run(testsuite('tests/'));

coverageReport = coverageResult.Result;
percentCovered = 100 * sum([coverageReport.LineCoverage]) / numel([coverageReport.LineCoverage]);

if percentCovered < 80
    error('CI:coverageBelowThreshold', 'Coverage %.1f%% is below the 80%% gate.', percentCovered);
end
```

A coverage gate blocks merges that drop test coverage below an agreed
threshold — not a guarantee of correctness (Level 3 Module 09 already
noted coverage isn't correctness), but a guardrail against silently
growing untested code as a project scales.

## Versioning and release automation

```yaml
      - name: Build compiled application
        uses: matlab-actions/run-command@v2
        with:
          command: |
            version = fileread('VERSION.txt');
            mcc('-m', 'src/myApp.m', '-o', ['MyApp_v' strtrim(version)]);

      - name: Create release
        if: startsWith(github.ref, 'refs/tags/v')
        uses: softprops/action-gh-release@v1
        with:
          files: build/**
```

Tagging a build with the source version (from a `VERSION.txt` or git
tag) ties every compiled artifact back to an exact commit — resolving
Module 05's "which build is actually running in production" problem at
the build-artifact level, not just in application logs.

## A complete pipeline shape

1. **On every push**: run static analysis (`checkcode`), then the full
   unit test suite with coverage, failing fast on either.
2. **On merge to main**: additionally build the compiled application
   (`mcc`) and archive the artifact.
3. **On a version tag**: publish the tagged, versioned build as a
   release artifact, optionally deploying to Production Server or a
   Web App Server target (Module 05).

## How It Actually Works

Running MATLAB unit tests (Module 09, Level 3) inside a CI pipeline means
invoking MATLAB **non-interactively** via `matlab -batch` (or the older
`-r`), which starts the full MATLAB engine — interpreter, JIT compiler,
toolbox licensing — headlessly, executes a specified script or command,
and exits with a process return code that CI systems use to determine
pass/fail; this is mechanically the same execution engine as an
interactive session (Module 01, Level 1's REPL model), just driven by a
script instead of a human typing into the Command Window, and it's why
CI-run tests can hit the exact same licensing and toolbox-availability
issues (Module 08, Level 2) that interactive sessions can if the CI
machine's license server or installed toolboxes don't match what the code
expects.

Test result artifacts (JUnit-style XML, code-coverage reports) are
generated by MATLAB's testing framework instrumenting the same
interpreter execution path the Profiler uses (Module 04) — code coverage
specifically works by tracking, at the statement or line level, which
lines actually executed during the test run, the same execution-tracking
hook mechanism the debugger (Module 09, Level 2) and profiler both rely
on, repurposed here to answer "was this line ever reached by any test"
rather than "how long did this line take."

Caching MATLAB Runtime/toolbox installations between CI runs matters
because MATLAB's startup itself is non-trivial (loading the interpreter,
initializing toolbox path caches, checking out licenses) — a CI pipeline
that reinstalls or re-licenses MATLAB from scratch on every run pays this
fixed startup cost repeatedly, which is why persistent build agents or
cached container images are the standard mitigation.

*Note: based on documented `matlab -batch` and CI-integration behavior;
no MATLAB installation or CI runner is available in this environment to
execute a pipeline directly.*

## Practice

1. Write the GitHub Actions workflow YAML for a project that must pass
   `checkcode` static analysis and the full test suite before allowing
   a pull request to merge, with test results published as a check.
2. Explain why `-batch` (rather than the interactive MATLAB GUI) is
   what makes MATLAB testing possible in a headless CI runner, and what
   `assertSuccess(results)` contributes beyond just calling `runtests`.
3. Design a coverage gate policy: should new code added in a pull
   request be held to a stricter coverage threshold than the project's
   historical average, and how might a pipeline enforce that
   distinction (hint: compare coverage reports on changed files only,
   not project-wide).
4. Sketch the stages of a pipeline that builds and tests a MATLAB
   Coder-generated C library (Module 02) as a CI stage separate from
   the MATLAB unit tests, and explain why the generated C code likely
   needs its own C-level test harness rather than reusing the MATLAB
   `matlab.unittest` suite directly.
