---
title: "Final Quiz"
description: "Test what you have learned across all four levels of MATLAB Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does MATLAB stand for and what is its core data type?",
    "options": [
      "Math Lab; scalars",
      "MATrix LABoratory; everything is an array (even scalars are 1x1)",
      "Matrix Language; strings",
      "Library; tables"
    ],
    "correctIndex": 1,
    "explanation": "Array-first design."
  },
  {
    "question": "What are the elementwise square sum and the matrix product of A = [1 2; 3 4]?\n```python\nA=[[1,2],[3,4]]\nprint(sum(x*x for r in A for x in r), [[sum(A[i][k]*A[k][j] for k in range(2)) for j in range(2)] for i in range(2)])\n```\n",
    "options": [
      "30 [[7, 10], [15, 22]]",
      "30 [[1, 4], [9, 16]]",
      "10 [[1, 4], [9, 16]]",
      "7 [[7, 10], [15, 22]]"
    ],
    "correctIndex": 0,
    "explanation": "A.*A is elementwise; A*A is the matrix product. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it."
  },
  {
    "question": "How do you solve A x = b in MATLAB?",
    "options": [
      "x = A\\b (not inv(A)*b)",
      "x = A/b",
      "x = b/A",
      "x = inv(A)*b"
    ],
    "correctIndex": 0,
    "explanation": "More accurate and faster."
  },
  {
    "question": "What's the difference between c(2) and c{2} for a cell array?",
    "options": [
      "Same",
      "Reverse",
      "None",
      "Parentheses return a cell; braces return its contents"
    ],
    "correctIndex": 3,
    "explanation": "Common bug."
  },
  {
    "question": "What does int8(200) give?",
    "options": [
      "200",
      "Error",
      "-56",
      "127: integer types saturate"
    ],
    "correctIndex": 3,
    "explanation": "No wraparound."
  },
  {
    "question": "Why preallocate arrays in loops?",
    "options": [
      "Syntax",
      "Style",
      "Law",
      "Growing arrays each iteration copies memory and is slow"
    ],
    "correctIndex": 3,
    "explanation": "zeros(n,1)."
  },
  {
    "question": "A loop computing sum((1:n).^2) is slow. What's the idiom?",
    "options": [
      "Vectorize with array operations",
      "More loops",
      "Recursion",
      "parfor always"
    ],
    "correctIndex": 0,
    "explanation": "Built-in array ops."
  },
  {
    "question": "What does Newton's method give for sqrt(2) after 3 steps from x0 = 1?\n```python\nx=1.0\nfor _ in range(3): x=x-(x*x-2)/(2*x)\nprint(round(x,6))\n```\n",
    "options": [
      "1.414216",
      "1.0",
      "2.0",
      "1.5"
    ],
    "correctIndex": 0,
    "explanation": "Quadratic convergence. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it."
  },
  {
    "question": "What's the trapezoid estimate of the integral of x^2 on [0,1] with 4 intervals?\n```python\nn=4;h=1/n\nys=[(i*h)**2 for i in range(n+1)]\nprint(round(h*(ys[0]/2+sum(ys[1:-1])+ys[-1]/2),5))\n```\n",
    "options": [
      "0.33333",
      "0.25",
      "0.5",
      "0.34375"
    ],
    "correctIndex": 3,
    "explanation": "Exact value is 1/3; trapz overestimates convex functions. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it."
  },
  {
    "question": "Fit a line to (0,1),(1,3),(2,5). What are intercept and slope?\n```python\nimport statistics as s\nx=[0,1,2];y=[1,3,5]\nmx,my=s.mean(x),s.mean(y)\nb=sum((a-mx)*(c-my) for a,c in zip(x,y))/sum((a-mx)**2 for a in x)\nprint(my-b*mx,b)\n```\n",
    "options": [
      "1.0 3.0",
      "3.0 2.0",
      "1.0 2.0",
      "0.0 1.0"
    ],
    "correctIndex": 2,
    "explanation": "polyfit(x,y,1) returns [2 1]. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it."
  },
  {
    "question": "Which solver handles stiff ODEs?",
    "options": [
      "ode23",
      "ode45",
      "fzero",
      "ode15s (ode45 is for non-stiff)"
    ],
    "correctIndex": 3,
    "explanation": "Implicit methods."
  },
  {
    "question": "An A-matrix is nearly singular and A\\b warns. What does it mean?",
    "options": [
      "Syntax",
      "Success",
      "The system is ill-conditioned: results are sensitive to small errors",
      "Plot"
    ],
    "correctIndex": 2,
    "explanation": "Check cond(A)."
  },
  {
    "question": "What does the Nyquist criterion require?",
    "options": [
      "Equal rate",
      "Sampling above twice the highest signal frequency to avoid aliasing",
      "Any rate",
      "Half rate"
    ],
    "correctIndex": 1,
    "explanation": "Anti-alias filter."
  },
  {
    "question": "What is the FFT bin spacing for 2000 samples at 1000 Hz?\n```python\nprint(1000/2000)\n```\n",
    "options": [
      "1.0",
      "2.0",
      "0.5",
      "1000"
    ],
    "correctIndex": 2,
    "explanation": "fs/N Hz per bin. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it."
  },
  {
    "question": "A parfor loop gives wrong results. What's the likely cause?",
    "options": [
      "Plots",
      "Too few workers",
      "Syntax",
      "Iterations depend on each other (parfor needs independent iterations)"
    ],
    "correctIndex": 3,
    "explanation": "Parallel Computing Toolbox."
  },
  {
    "question": "Data is bigger than memory. What do you use?",
    "options": [
      "load everything",
      "cellfun",
      "datastore and tall arrays (or chunked reads)",
      "Bigger loops"
    ],
    "correctIndex": 2,
    "explanation": "Out-of-memory analysis."
  },
  {
    "question": "What does a handle class give you?",
    "options": [
      "Value copies",
      "Immutability",
      "Reference semantics: copies of the variable refer to the same object",
      "Static methods"
    ],
    "correctIndex": 2,
    "explanation": "Value classes copy on assignment."
  },
  {
    "question": "How do you compare floating-point values in unit tests?",
    "options": [
      "strcmp",
      "==",
      "isequal",
      "Within a tolerance (AbsTol/RelTol), not =="
    ],
    "correctIndex": 3,
    "explanation": "Round-off."
  },
  {
    "question": "What do you use to deploy to users without a MATLAB license?",
    "options": [
      "Email",
      "Copy .m files",
      "MATLAB Compiler with the free MATLAB Runtime",
      "Zip"
    ],
    "correctIndex": 2,
    "explanation": "Standalone apps."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Root finding and integration

Implement newton, bisect (with a bracketing check) and a composite trapz, then compare their behaviour on x^2 - 2 and on integrals of x^2 and sin. Python mirrors the MATLAB functions fzero/trapz.

**Starter code**

```matlab
import math

def newton(f, df, x0, tol=1e-10, max_iter=50):
    # iterate x -= f(x)/df(x); return (x, iterations) once |step| < tol; RuntimeError if it never converges
    return x0, 0

def bisect(f, a, b, tol=1e-8):
    # halve [a, b] keeping the sign change; ValueError("root not bracketed") if f(a)*f(b) > 0;
    # return (midpoint, number of halvings) once the half-width <= tol
    return a, 0

def trapz(f, a, b, n):
    # composite trapezoid rule with n intervals
    return 0.0

f = lambda x: x * x - 2
r, it = newton(f, lambda x: 2 * x, 1.0)
print("newton:", round(r, 8), "iterations", it)
r, it = bisect(f, 0, 2)
print("bisect:", round(r, 6), "iterations", it)
try:
    bisect(f, 2, 3)
except ValueError as e:
    print("error:", e)
print("trapz n=4:", round(trapz(lambda x: x * x, 0, 1, 4), 5), " n=100:", round(trapz(lambda x: x * x, 0, 1, 100), 6))
print("sin integral:", round(trapz(math.sin, 0, math.pi, 200), 5))
```

**Expected output**

```text
newton: 1.41421356 iterations 5
bisect: 1.414214 iterations 27
error: root not bracketed
trapz n=4: 0.34375  n=100: 0.33335
sin integral: 1.99996
```

<details><summary>Show solution and explanation</summary>

<pre><code>import math

def newton(f, df, x0, tol=1e-10, max_iter=50):
    x = x0
    for i in range(1, max_iter + 1):
        step = f(x) / df(x)
        x -= step
        if abs(step) &lt; tol:
            return x, i
    raise RuntimeError(&quot;did not converge&quot;)

def bisect(f, a, b, tol=1e-8):
    if f(a) * f(b) &gt; 0:
        raise ValueError(&quot;root not bracketed&quot;)
    n = 0
    while (b - a) / 2 &gt; tol:
        m = (a + b) / 2
        if f(a) * f(m) &lt;= 0:
            b = m
        else:
            a = m
        n += 1
    return (a + b) / 2, n

def trapz(f, a, b, n):
    h = (b - a) / n
    return h * (f(a) / 2 + sum(f(a + i * h) for i in range(1, n)) + f(b) / 2)

f = lambda x: x * x - 2
r, it = newton(f, lambda x: 2 * x, 1.0)
print(&quot;newton:&quot;, round(r, 8), &quot;iterations&quot;, it)
r, it = bisect(f, 0, 2)
print(&quot;bisect:&quot;, round(r, 6), &quot;iterations&quot;, it)
try:
    bisect(f, 2, 3)
except ValueError as e:
    print(&quot;error:&quot;, e)
print(&quot;trapz n=4:&quot;, round(trapz(lambda x: x * x, 0, 1, 4), 5), &quot; n=100:&quot;, round(trapz(lambda x: x * x, 0, 1, 100), 6))
print(&quot;sin integral:&quot;, round(trapz(math.sin, 0, math.pi, 200), 5))
</code></pre>

<p>Newton converges in a handful of iterations when started near the root, bisection needs about 27 halvings for 1e-8 but always works when the root is bracketed, and the trapezoid rule&#x27;s error shrinks as the interval count grows. Verified by running the solution in Python (an equivalent of the MATLAB approach).</p>
</details>

### Linear solve and least-squares fit

Implement Gaussian elimination with partial pivoting (what A\b does for square systems) and a least-squares polyfit using the normal equations. Detect singular matrices, fit a noisy line and compute R^2.

**Starter code**

```matlab
def solve(A, b):
    # Gaussian elimination with partial pivoting; ValueError("matrix is singular") when a pivot is ~0
    return []

def polyfit(xs, ys, deg):
    # least squares via the normal equations (X'X)c = X'y (use solve); coefficients highest power first
    return []

print([round(v, 6) for v in solve([[2, 1, -1], [-3, -1, 2], [-2, 1, 2]], [8, -11, -3])])
try:
    solve([[1, 2], [2, 4]], [1, 2])
except ValueError as e:
    print("error:", e)
xs = [0, 1, 2, 3, 4]
ys = [1.1, 2.9, 5.2, 6.8, 9.1]
c = polyfit(xs, ys, 1)
print("line coefficients:", [round(v, 4) for v in c])
pred = [c[0] * x + c[1] for x in xs]
ss_res = sum((y - p) ** 2 for y, p in zip(ys, pred))
mean = sum(ys) / len(ys)
print("R^2:", round(1 - ss_res / sum((y - mean) ** 2 for y in ys), 4))
print("quadratic exact:", [round(v, 6) + 0.0 for v in polyfit([0, 1, 2, 3], [1, 2, 5, 10], 2)])
```

**Expected output**

```text
[2.0, 3.0, -1.0]
error: matrix is singular
line coefficients: [1.99, 1.04]
R^2: 0.9973
quadratic exact: [1.0, 0.0, 1.0]
```

<details><summary>Show solution and explanation</summary>

<pre><code>def solve(A, b):
    &quot;&quot;&quot;Gaussian elimination with partial pivoting (what A\\b does for a square system).&quot;&quot;&quot;
    n = len(A)
    M = [row[:] + [b[i]] for i, row in enumerate(A)]
    for c in range(n):
        p = max(range(c, n), key=lambda r: abs(M[r][c]))
        if abs(M[p][c]) &lt; 1e-12:
            raise ValueError(&quot;matrix is singular&quot;)
        M[c], M[p] = M[p], M[c]
        for r in range(c + 1, n):
            f = M[r][c] / M[c][c]
            for k in range(c, n + 1):
                M[r][k] -= f * M[c][k]
    x = [0.0] * n
    for i in range(n - 1, -1, -1):
        x[i] = (M[i][n] - sum(M[i][j] * x[j] for j in range(i + 1, n))) / M[i][i]
    return x

def polyfit(xs, ys, deg):
    &quot;&quot;&quot;Least squares via the normal equations (X&#x27;X)c = X&#x27;y; coefficients highest power first.&quot;&quot;&quot;
    X = [[x ** p for p in range(deg, -1, -1)] for x in xs]
    XtX = [[sum(X[k][i] * X[k][j] for k in range(len(xs))) for j in range(deg + 1)] for i in range(deg + 1)]
    Xty = [sum(X[k][i] * ys[k] for k in range(len(xs))) for i in range(deg + 1)]
    return solve(XtX, Xty)

print([round(v, 6) for v in solve([[2, 1, -1], [-3, -1, 2], [-2, 1, 2]], [8, -11, -3])])
try:
    solve([[1, 2], [2, 4]], [1, 2])
except ValueError as e:
    print(&quot;error:&quot;, e)
xs = [0, 1, 2, 3, 4]
ys = [1.1, 2.9, 5.2, 6.8, 9.1]
c = polyfit(xs, ys, 1)
print(&quot;line coefficients:&quot;, [round(v, 4) for v in c])
pred = [c[0] * x + c[1] for x in xs]
ss_res = sum((y - p) ** 2 for y, p in zip(ys, pred))
mean = sum(ys) / len(ys)
print(&quot;R^2:&quot;, round(1 - ss_res / sum((y - mean) ** 2 for y in ys), 4))
print(&quot;quadratic exact:&quot;, [round(v, 6) + 0.0 for v in polyfit([0, 1, 2, 3], [1, 2, 5, 10], 2)])
</code></pre>

<p>A\b is elimination with pivoting; polyfit minimizes squared residuals, and R^2 compares residuals with the variance of the data. Production code prefers QR for numerical stability. Verified by running the solution in Python (an equivalent of the MATLAB approach).</p>
</details>
