# 06 · Plotting & Visualization Basics

MATLAB's plotting is one of its strongest, most heavily used features —
built-in, fast, and designed so a single `plot()` call produces a usable
figure with almost no configuration.

## The basic `plot()`

```matlab
x = 0:0.1:2*pi;
y = sin(x);
plot(x, y)
```

This opens a **Figure window** showing a smooth sine curve. `x` and `y`
must be the same length — `plot()` connects them point by point in order.
Nothing is printed to the Command Window; the output is entirely visual.

## Labels, title, and grid — always add these

A plot without axis labels is close to useless for anyone but the person
who made it five minutes ago:

```matlab
x = 0:0.1:2*pi;
y = sin(x);
plot(x, y)
xlabel('x (radians)')
ylabel('sin(x)')
title('Sine Wave')
grid on
```

`grid on` overlays light gridlines, which make it much easier to read
values off the curve. These four lines — `plot`, `xlabel`, `ylabel`,
`title` (plus usually `grid on`) — are the baseline for almost every
MATLAB plot you'll make.

## Multiple lines on one plot

```matlab
x = 0:0.1:2*pi;
plot(x, sin(x), x, cos(x))
legend('sin(x)', 'cos(x)')
xlabel('x')
ylabel('y')
title('Sine and Cosine')
```

Passing multiple `x, y` pairs to one `plot()` call draws them all on the
same axes, each with a different default color, and `legend()` labels them
in the order they were plotted.

`hold on` is the alternative approach — useful when the curves come from
separate `plot()` calls (e.g. inside a loop) rather than one call:

```matlab
plot(x, sin(x))
hold on               % keep the current plot; don't erase it for the next one
plot(x, cos(x))
hold off               % (optional) return to default overwrite behavior
legend('sin(x)', 'cos(x)')
```

!!! warning "Forgetting `hold on`"
    Without `hold on`, each new `plot()` call **erases** the previous
    figure content and starts fresh — a very common beginner mistake when
    trying to build up a chart across multiple calls or a loop, where only
    the last curve ends up visible.

## Line styles, colors, and markers

A short format string as the third argument to `plot()` controls
appearance:

```matlab
plot(x, sin(x), 'r--')      % red dashed line
plot(x, sin(x), 'g-o')      % green solid line with circle markers
plot(x, sin(x), 'b:', 'LineWidth', 2)   % blue dotted, thicker line (name-value pair)
```

| Code | Meaning | Code | Meaning |
|------|---------|------|---------|
| `r` | red | `-` | solid line |
| `g` | green | `--` | dashed line |
| `b` | blue | `:` | dotted line |
| `k` | black | `-.` | dash-dot line |
| `o` | circle marker | `x` | x marker |

Format-string characters can combine freely (`'r--o'` = red, dashed, with
circle markers); properties that don't have a short code, like `LineWidth`
or `MarkerSize`, are set as trailing `'Name', value` pairs.

## Scatter plots

`scatter()` is the right choice for unconnected, discrete data points —
where drawing a line between them (as `plot` does) would misleadingly imply
an ordering or trend:

```matlab
hours_studied = [1, 2, 3, 4, 5, 6, 7, 8];
exam_score    = [52, 58, 63, 70, 74, 82, 88, 91];
scatter(hours_studied, exam_score, 'filled')
xlabel('Hours Studied')
ylabel('Exam Score')
title('Study Time vs. Score')
grid on
```

## Bar charts and histograms

```matlab
categories = {'Q1', 'Q2', 'Q3', 'Q4'};
revenue    = [120, 150, 90, 200];
bar(revenue)
set(gca, 'XTickLabel', categories)   % label the bars with category names
ylabel('Revenue ($k)')
title('Quarterly Revenue')
```

`bar()` is for a small number of discrete categories with known values.
`histogram()` is different: it takes a set of raw, possibly-many
observations and **automatically bins them**, showing you the underlying
distribution:

```matlab
data = randn(1, 1000);     % 1000 random values from a standard normal distribution
histogram(data)
xlabel('Value')
ylabel('Count')
title('Distribution of Random Data')
```

## Multiple plots in one figure: `subplot`

```matlab
x = 0:0.1:2*pi;

subplot(2, 1, 1)      % 2 rows, 1 column, this is plot #1
plot(x, sin(x))
title('Sine')

subplot(2, 1, 2)      % 2 rows, 1 column, this is plot #2
plot(x, cos(x))
title('Cosine')
```

`subplot(rows, cols, index)` divides the current figure into a grid and
selects one cell to draw into next; `index` counts left-to-right,
top-to-bottom, same convention as reading English text.

## Saving a figure to a file

```matlab
plot(x, sin(x))
xlabel('x'); ylabel('sin(x)'); title('Sine Wave')
saveas(gcf, 'sine_wave.png')     % gcf = "get current figure"
```

`saveas` infers the file format from the extension (`.png`, `.jpg`, `.pdf`,
`.fig` for MATLAB's own editable format, etc.). `exportgraphics(gcf,
'sine_wave.png', 'Resolution', 300)` is the more modern, higher-quality
alternative when you need print-resolution output.

## New vs. reused figures

```matlab
figure               % opens a brand-new figure window
plot(x, sin(x))

figure               % opens ANOTHER new window, doesn't touch the first
plot(x, cos(x))
```

Without an explicit `figure` call, MATLAB reuses the current figure window
(and, per the `hold on`/`hold off` rule above, overwrites its contents by
default) — call `figure` explicitly whenever you want a fresh, separate
window.

## Plotting cheat sheet

| Task | Function |
|------|----------|
| Line plot | `plot(x, y)` |
| Scatter (unconnected points) | `scatter(x, y)` |
| Bar chart (categories) | `bar(values)` |
| Histogram (distribution of raw data) | `histogram(data)` |
| Axis labels / title | `xlabel()`, `ylabel()`, `title()` |
| Multiple series, one legend | `legend('name1', 'name2')` |
| Keep adding to same axes | `hold on` / `hold off` |
| Gridlines | `grid on` |
| New figure window | `figure` |
| Multiple plots, one figure | `subplot(rows, cols, index)` |
| Save to file | `saveas(gcf, 'name.png')` |

## Exercise

Create `x = linspace(0, 10, 100)` and plot `y1 = x.^2` and `y2 = 10*x` on
the same axes using `hold on`, with a legend distinguishing the two curves,
axis labels, a title, and `grid on`. Then use `subplot` to create a 1×2
figure: the left panel showing your line plot from above, and the right
panel showing a `histogram` of 500 values from `randn(1, 500)`. Save the
full figure to `exercise_plot.png` using `saveas`.
