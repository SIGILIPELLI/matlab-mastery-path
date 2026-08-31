# 09 · Testing MATLAB Code

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. The MATLAB Unit Testing Framework's classes, decorators, and
    runner behaviors are documented, version-stable features,
    hand-traced against the MATLAB documentation rather than executed
    in MATLAB itself.

Every module so far has trusted that hand-written functions behave
correctly. MATLAB's Unit Testing Framework formalizes that trust into
automated, repeatable tests — essential once code grows past a few
functions, or before refactoring anything you can't afford to silently
break.

## Script-based tests: the quick path

```matlab
% file: test_addTwo.m
function tests = test_addTwo
    tests = functiontests(localfunctions);
end

function testPositiveNumbers(testCase)
    result = addTwo(3, 4);
    verifyEqual(testCase, result, 7);
end

function testNegativeNumbers(testCase)
    result = addTwo(-3, -4);
    verifyEqual(testCase, result, -7);
end

function testZero(testCase)
    result = addTwo(0, 0);
    verifyEqual(testCase, result, 0);
end
```

```matlab
function result = addTwo(a, b)
    result = a + b;
end
```

Running `runtests('test_addTwo')` discovers every local function
prefixed by MATLAB's convention (functions returned by `localfunctions`
that accept a single `testCase` argument), executes each in isolation,
and reports pass/fail with a summary.

```matlab
results = runtests('test_addTwo');
disp(results);
% Totals: 3 Passed, 0 Failed, 0 Incomplete
```

## Class-based tests: the scalable path

For larger suites, a `matlab.unittest.TestCase` subclass gives structure
— shared setup/teardown, test parameterization, and organization into
one file per unit under test:

```matlab
classdef BankAccountTest < matlab.unittest.TestCase

    properties
        Account
    end

    methods (TestMethodSetup)
        function createAccount(testCase)
            testCase.Account = BankAccount("Test User", 100);
        end
    end

    methods (Test)

        function testInitialBalance(testCase)
            testCase.verifyEqual(testCase.Account.Balance, 100);
        end

        function testDeposit(testCase)
            acc = testCase.Account.deposit(50);
            testCase.verifyEqual(acc.Balance, 150);
        end

        function testWithdrawSufficientFunds(testCase)
            acc = testCase.Account.withdraw(30);
            testCase.verifyEqual(acc.Balance, 70);
        end

        function testWithdrawInsufficientFundsThrows(testCase)
            testCase.verifyError(@() testCase.Account.withdraw(1000), ...
                'BankAccount:insufficientFunds');
        end

        function testDepositNegativeThrows(testCase)
            testCase.verifyError(@() testCase.Account.deposit(-10), ...
                'BankAccount:invalidAmount');
        end

    end
end
```

`TestMethodSetup` runs before *every* test method, giving each test a
fresh `Account` — tests must not depend on execution order or leak
state between each other. `TestClassSetup` (run once per class, not per
test) is the place for expensive shared setup, like loading a large
fixture file.

```matlab
methods (TestClassSetup)
    function loadSharedFixture(testCase)
        testCase.SharedData = load('large_fixture.mat');
    end
end
```

## Verification vs. assertion vs. assumption

```matlab
function testExample(testCase)
    testCase.verifyEqual(computeValue(), 42);   % failure recorded, test CONTINUES
    testCase.assertEqual(size(x), [3 3]);       % failure STOPS this test immediately
    testCase.assumeTrue(hasLicense('Optimization_Toolbox'));  % failure SKIPS this test (not a failure)
end
```

- **`verify*`** records a failure but lets the rest of the test method
  run — use for independent checks where seeing all of them matters.
- **`assert*`** stops the test immediately on failure — use when a later
  line would error or be meaningless without the assertion holding (e.g.
  don't index into a matrix whose size assertion just failed).
- **`assume*`** skips the test (not counted as failed) when a
  precondition isn't met — use for environment-dependent tests (toolbox
  license, platform-specific behavior) that shouldn't count as failures
  when the precondition legitimately doesn't hold.

## Common qualification methods

```matlab
testCase.verifyEqual(actual, expected);
testCase.verifyEqual(actual, expected, 'AbsTol', 1e-10);   % floating-point comparison
testCase.verifyTrue(condition);
testCase.verifyGreaterThan(value, threshold);
testCase.verifyError(@() riskyFunction(), 'MyPkg:myError');
testCase.verifyWarning(@() deprecatedFunction(), 'MyPkg:deprecated');
testCase.verifyClass(obj, 'BankAccount');
testCase.verifyEmpty(result);
testCase.verifySize(matrix, [3 4]);
```

`'AbsTol'`/`'RelTol'` name-value pairs on `verifyEqual` matter for any
numeric result from floating-point computation — exact equality on
`double` results of arithmetic is rarely safe to assert (see Level 1's
notes on floating-point representation).

## Parameterized tests

```matlab
classdef MathTest < matlab.unittest.TestCase

    properties (TestParameter)
        inputValue = {1, 4, 9, 16, 25};
    end

    methods (Test)
        function testSqrtIsNonNegative(testCase, inputValue)
            result = sqrt(inputValue);
            testCase.verifyGreaterThanOrEqual(result, 0);
        end
    end

end
```

A `TestParameter` property makes the framework run the decorated test
once per value automatically — five separate test results here, one per
`inputValue` entry, rather than one test looping internally (which would
hide which specific input failed).

## Mocking and test doubles

```matlab
function testProcessOrderCallsPaymentGateway(testCase)
    mockGateway = testCase.createMock(?PaymentGateway);
    testCase.assignOutputsWhen(withExactInputs(mockGateway.charge(100, 'USD')), true);

    processOrder(mockGateway, 100, 'USD');

    testCase.verifyCalled(mockGateway.charge(100, 'USD'));
end
```

The Mocking Framework (`matlab.mock`) lets tests substitute a fake
`PaymentGateway` for the real one, so `processOrder`'s logic can be
tested without actually calling a real payment API — the mock records
calls and can assert exactly what was invoked with what arguments.

## Organizing and running a suite

```matlab
suite = testsuite('tests/');           % discover every test in a folder, recursively
runner = matlab.unittest.TestRunner.withTextOutput('Verbosity', matlab.unittest.Verbosity.Detailed);
results = runner.run(suite);

table(results)     % tabular pass/fail/duration summary
```

```matlab
% Add code coverage reporting
import matlab.unittest.plugins.CodeCoveragePlugin
runner.addPlugin(CodeCoveragePlugin.forFolder('src/', ...
    'Producing', matlab.unittest.plugins.codecoverage.CoberturaFormat('coverage.xml')));
results = runner.run(suite);
```

Coverage plugins report which lines of `src/` were exercised by the
suite — a useful signal for finding untested branches, though 100%
coverage doesn't guarantee correctness (only that every line *ran*, not
that every case was checked meaningfully).

## Test-driven development in practice

The natural TDD loop with this framework:

1. Write a failing test describing the behavior you want:
   `testWithdrawInsufficientFundsThrows` before `withdraw` even checks
   the balance.
2. Run it, watch it fail (`runtests` reports the failure — confirms the
   test actually exercises the intended path).
3. Write the minimum code to pass (`if amount > obj.Balance error(...)
   end`).
4. Refactor with confidence, re-running the suite after each change to
   catch regressions immediately.

## Practice

1. Write a class-based test suite for a `Stack` class (`push`, `pop`,
   `peek`, `isEmpty`) covering: pushing then popping returns LIFO order,
   popping an empty stack throws a specific error identifier, and
   `isEmpty` is true only when nothing has been pushed or everything
   pushed has been popped.
2. Convert one test to use `TestParameter` to check `isEmpty` behavior
   across several sequences of push/pop operations rather than one
   hardcoded sequence.
3. Explain, with a concrete example test method, a scenario where using
   `assertEqual` instead of `verifyEqual` matters — where continuing
   execution after the failed check would itself throw an unrelated
   error.
4. Sketch a `TestClassSetup` vs `TestMethodSetup` split for a suite that
   tests parsing of a large shared reference file — which setup should
   load the file once, and which should reset any per-test parser
   state, and why mixing them up would either slow the suite down or
   cause tests to leak state into each other.
