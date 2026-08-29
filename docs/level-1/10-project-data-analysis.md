# 10 · Project — Data Analysis & Plotting Script

Time to combine everything from this level into one working script: reading
data, computing statistics, filtering with logical indexing, writing a
function, and producing a labeled plot.

!!! note "Verification note"
    MATLAB was not available in this environment. Every number below was
    hand-computed (mean, sample standard deviation with `n-1` denominator,
    sorted order, threshold filter) and cross-checked with an independent
    Python calculation using the same formulas MATLAB's `mean`/`std`
    functions document — not executed in MATLAB itself.

## The scenario

You have exam scores for 10 students and want a script that: loads the
data, reports summary statistics, flags who passed (score ≥ 70), and
produces a plot showing the distribution with the passing threshold marked.

## Step 1 — the data

```matlab
scores = [72, 88, 95, 61, 79, 84, 90, 55, 68, 99];
students = {'Ana', 'Ben', 'Cara', 'Dev', 'Ella', ...
            'Finn', 'Gia', 'Hao', 'Ivy', 'Jai'};
```

(This mirrors what `readtable` would give you from a real CSV — the
project works the same way whether the data is typed in directly or loaded
from a file, which is exactly the point of Module 07.)

## Step 2 — a reusable statistics function

```matlab
function report = analyze_scores(scores)
    report.mean_score = mean(scores);
    report.std_score  = std(scores);
    report.max_score  = max(scores);
    report.min_score  = min(scores);
    report.sorted     = sort(scores);
end
```

```matlab
>> r = analyze_scores(scores)
r =

  struct with fields:

    mean_score: 79.1000
     std_score: 14.7305
     max_score: 99
     min_score: 55
        sorted: [55 61 68 72 79 84 88 90 95 99]
```

`std()` uses `n-1` in the denominator by default (the sample standard
deviation, Bessel's correction) — the right choice when your data is a
sample rather than the entire population, and MATLAB's documented default.

## Step 3 — filtering with logical indexing

```matlab
passing_mask = scores >= 70;
passing_scores = scores(passing_mask);
passing_names = students(passing_mask);   % cell arrays support logical indexing too

fprintf('%d of %d students passed (%.0f%%)\n', ...
    sum(passing_mask), length(scores), 100 * mean(passing_mask));
```

```
7 of 10 students passed (70%)
```

`mean(passing_mask)` works because `passing_mask` is a `logical` array —
MATLAB treats `true`/`false` as `1`/`0` in arithmetic, so the mean of a
logical array is exactly the fraction that's `true`, a common one-line
idiom for "what percentage passed."

```matlab
>> passing_scores
passing_scores =

    72    88    95    79    84    90    99

>> passing_names
passing_names =

  1x7 cell array

    {'Ana'}    {'Ben'}    {'Cara'}    {'Ella'}    {'Finn'}    {'Gia'}    {'Jai'}
```

## Step 4 — the plot

```matlab
figure
bar(scores)
hold on
yline(70, 'r--', 'Passing Threshold', 'LineWidth', 2);
set(gca, 'XTickLabel', students, 'XTick', 1:length(students))
xlabel('Student')
ylabel('Score')
title(sprintf('Exam Scores (Mean: %.1f, Pass rate: %.0f%%)', ...
    mean(scores), 100 * mean(passing_mask)))
grid on
saveas(gcf, 'exam_scores_report.png')
```

`yline(70, ...)` draws a horizontal reference line at `y = 70` across the
whole plot, with an inline label — a clean way to show a threshold against
bar or line data without manually plotting a second series.

## Step 5 — the complete script

```matlab
% exam_analysis.m
scores = [72, 88, 95, 61, 79, 84, 90, 55, 68, 99];
students = {'Ana', 'Ben', 'Cara', 'Dev', 'Ella', ...
            'Finn', 'Gia', 'Hao', 'Ivy', 'Jai'};

report = analyze_scores(scores);
fprintf('Mean: %.2f | Std: %.2f | Max: %d | Min: %d\n', ...
    report.mean_score, report.std_score, report.max_score, report.min_score);

passing_mask = scores >= 70;
fprintf('%d of %d students passed (%.0f%%)\n', ...
    sum(passing_mask), length(scores), 100 * mean(passing_mask));

figure
bar(scores)
hold on
yline(70, 'r--', 'Passing Threshold', 'LineWidth', 2);
set(gca, 'XTickLabel', students, 'XTick', 1:length(students))
xlabel('Student'); ylabel('Score')
title(sprintf('Exam Scores (Mean: %.1f, Pass rate: %.0f%%)', ...
    mean(scores), 100 * mean(passing_mask)))
grid on
saveas(gcf, 'exam_scores_report.png')

function report = analyze_scores(scores)
    report.mean_score = mean(scores);
    report.std_score  = std(scores);
    report.max_score  = max(scores);
    report.min_score  = min(scores);
    report.sorted     = sort(scores);
end
```

Running `exam_analysis` in the Command Window prints the summary lines,
opens a labeled bar chart with a threshold line, and saves it to a PNG —
one script covering statistics, logical filtering, functions, and
plotting, the full arc of this level.

## What this project used from every module

| Module | Used here as |
|--------|---------------|
| 01 · What Is MATLAB? | Script structure, running a `.m` file |
| 02 · Variables & Types | `scores` (double array), `students` (cell of char) |
| 03 · Vectors & Matrix Ops | `scores >= 70`, `mean()`, `sort()` |
| 04 · Control Flow | (implicit — logical indexing replaces an explicit loop) |
| 05 · Functions | `analyze_scores`, returning a struct |
| 06 · Plotting | `bar`, `yline`, labels, `saveas` |
| 07 · Data Files | Same shape as data from `readtable` |
| 08 · String Processing | `sprintf`/`fprintf` for the report messages |
| 09 · Numerical Methods | `mean`/`std` as summary statistics |

## Exercise

Extend `exam_analysis.m`: add a second local function,
`grade_letter(score)`, that returns `'A'` for ≥ 90, `'B'` for ≥ 80, `'C'`
for ≥ 70, and `'F'` otherwise (using `if`/`elseif` from Module 04). Loop
over `scores` with a `for` loop, building a cell array `grades` of each
student's letter grade, and add a column of grade labels above each bar
using `text()` at each bar's x/y position. Save the final annotated figure
as `exam_scores_with_grades.png`.
