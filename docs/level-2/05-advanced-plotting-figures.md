# 05 · Advanced Plotting & Figures

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Plotting function signatures, handle-graphics property names, and
    subplot/tiledlayout behavior are documented in MATLAB's Graphics
    documentation and were hand-traced against it, not executed in
    MATLAB itself.

Level 1 covered `plot`, `xlabel`, `title`, and saving a figure. This
module goes into the handle-graphics object model underneath every plot,
multi-panel layouts, and the plot types you reach for once a simple line
plot doesn't say enough.

## Everything is an object with a handle

Every graphics element MATLAB draws — figure, axes, line, text — is an
object, and every plotting function returns a **handle** to the object it
created:

```matlab
x = linspace(0, 10, 100);
fig = figure;                  % handle to the figure window
ax = axes;                     % handle to axes within it
h = plot(ax, x, sin(x));       % handle to the line object itself
```

Once you hold a handle, you can query or change any property of that
object after the fact, instead of only through plotting-call arguments:

```matlab
h.Color = [0.85 0.33 0.10];    % RGB triplet, orange
h.LineWidth = 2;
h.LineStyle = '--';
set(ax, 'FontSize', 12, 'XGrid', 'on');
```

`get(h)` lists every settable property on an object — invaluable when you
know roughly what you want ("thicker line", "dashed") but not the exact
property name.

### `gcf`, `gca`, `gco`

When you don't hold a handle explicitly (e.g. inside a script that called
bare `plot(x,y)`), three functions retrieve the "current" object:

```matlab
gcf   % get current figure
gca   % get current axes
gco   % get current object (e.g. last clicked)
```

`gca` is the workhorse — `xlabel(...)`, `title(...)`, etc. without an
axes argument implicitly target `gca`; passing a handle explicitly (as
above with `ax`) is preferred in any function or script that might create
multiple figures, since `gca` silently targeting the wrong axes is a
common plotting bug.

## Multi-panel figures: `tiledlayout` (preferred) vs `subplot`

`subplot(rows, cols, index)` is the classic way to place multiple axes in
one figure:

```matlab
figure;
subplot(2, 2, 1); plot(x, sin(x));   title('sin');
subplot(2, 2, 2); plot(x, cos(x));   title('cos');
subplot(2, 2, 3); plot(x, tan(x));   title('tan'); ylim([-10 10]);
subplot(2, 2, 4); plot(x, x.^2);     title('x^2');
```

`tiledlayout` (R2019b+) is the modern replacement, with better control
over spacing and a cleaner API for shared labels/titles:

```matlab
figure;
t = tiledlayout(2, 2, 'TileSpacing', 'compact', 'Padding', 'compact');
nexttile; plot(x, sin(x));  title('sin');
nexttile; plot(x, cos(x));  title('cos');
nexttile; plot(x, tan(x));  title('tan'); ylim([-10 10]);
nexttile; plot(x, x.^2);    title('x^2');

title(t, 'Four Basic Functions');   % one shared title for the whole layout
xlabel(t, 'x');                     % one shared x-label
```

`tiledlayout`'s big advantage: `title(t, ...)` and `xlabel(t, ...)` set a
label *once* for the whole grid instead of repeating it in every
`subplot` panel, and tiles can span multiple grid cells with
`nexttile([2 1])` (span 2 rows, 1 column) — something `subplot` can't do
cleanly.

## Plot types beyond `plot`

| Function | Use case |
|---|---|
| `scatter(x, y)` | Discrete points, optionally sized/colored per point |
| `bar(categories, values)` | Categorical comparisons |
| `histogram(data)` | Distribution of a single variable |
| `stairs(x, y)` | Step functions / discrete-time signals |
| `stem(x, y)` | Discrete sequences (common in signal processing) |
| `surf(X, Y, Z)` / `mesh(X, Y, Z)` | 3D surfaces over a grid |
| `contour(X, Y, Z)` | 2D contour lines of a scalar field |
| `errorbar(x, y, err)` | Data with uncertainty bars |

```matlab
data = randn(1, 1000);       % NOTE: randn output not verifiable without
                              % running MATLAB; shown for the API pattern.
figure;
histogram(data, 30, 'Normalization', 'pdf');
hold on;
xs = linspace(-4, 4, 200);
plot(xs, exp(-xs.^2/2)/sqrt(2*pi), 'r-', 'LineWidth', 2);
legend('sample histogram', 'standard normal pdf');
```

`scatter` with per-point size and color encodes two extra data
dimensions on a 2D plot:

```matlab
n = 50;
x = 1:n; y = (1:n) + 5*sin((1:n)/3);
sizes = 20 + 5*(1:n);
colors = (1:n);
scatter(x, y, sizes, colors, 'filled');
colorbar;   % shows the color -> value mapping
```

## 3D surfaces: `meshgrid` + `surf`

```matlab
[X, Y] = meshgrid(-3:0.2:3, -3:0.2:3);
Z = X .* exp(-X.^2 - Y.^2);

figure;
surf(X, Y, Z);
shading interp;        % smooth face coloring instead of flat facets
colormap(parula);
colorbar;
xlabel('x'); ylabel('y'); zlabel('z');
view(-30, 45);          % azimuth, elevation for the 3D camera
```

`shading interp` removes the visible grid lines between facets by
interpolating color across each patch — the difference between a
"blocky" and a "smooth" surface plot with identical underlying data.
`view(az, el)` sets the camera angle; `view(2)` snaps to a top-down 2D
view (equivalent to `contour`'s viewpoint) which is a quick way to
sanity-check a `surf` plot against a `contour` of the same data.

## Annotations and legends

```matlab
figure;
plot(x, sin(x), 'b-', x, cos(x), 'r--');
legend('sin(x)', 'cos(x)', 'Location', 'northeast');
text(pi, 0, '\pi', 'FontSize', 14, 'HorizontalAlignment', 'center');
annotation('arrow', [0.3 0.4], [0.6 0.5]);
```

`text(x, y, str)` places a label at data coordinates; `annotation`
instead uses normalized *figure* coordinates (0 to 1 across the whole
figure, independent of axes limits) — useful for callouts that shouldn't
move if the axes limits change. MATLAB's `text`/`title`/`xlabel` strings
support a subset of TeX/LaTeX markup by default (`\pi`, `^{}`, `_{}`,
Greek letters) without any special flag.

## Saving figures for publication

```matlab
exportgraphics(gcf, 'figure1.png', 'Resolution', 300);
exportgraphics(gcf, 'figure1.pdf', 'ContentType', 'vector');
```

`exportgraphics` (R2020a+) is the modern, recommended replacement for the
older `saveas`/`print` combination — it crops whitespace sensibly by
default and cleanly distinguishes raster (`'Resolution'`, for `.png`/
`.jpg`) from vector (`'ContentType','vector'`, for `.pdf`/`.eps`) output.
For a multi-page or figure-per-tile export, loop over figures/tiles and
call `exportgraphics` once per file, since a single call exports one
target.

## Colormaps and colorbars

```matlab
colormap(turbo);     % or: parula (default), jet, viridis-like 'turbo', gray
c = colorbar;
c.Label.String = 'Temperature (°C)';
clim([0 100]);        % explicit color-axis limits (renamed from caxis in R2022a+)
```

Choosing a colormap matters for correctness, not just aesthetics:
`parula` and `turbo` are perceptually more uniform than the legacy `jet`
colormap, meaning equal numeric differences look like roughly equal color
differences — `jet`'s sharp perceptual jumps can visually exaggerate or
hide features that aren't actually there in the underlying data.

## Interactive/live updates: redrawing efficiently

For an animation or a live-updating plot, avoid calling `plot` repeatedly
in a loop (which recreates the whole line object every frame) — update
the existing line's data instead:

```matlab
h = plot(NaN, NaN);   % create an empty line object once
xlim([0 10]); ylim([-1 1]);
for k = 1:100
    t = 0:0.01:k/10;
    set(h, 'XData', t, 'YData', sin(t));
    drawnow;                 % force the figure to render this frame now
end
```

`set(h, 'XData', ..., 'YData', ...)` mutates the existing line object in
place; `drawnow` flushes the graphics event queue so the change is
visible immediately rather than batched until the script finishes. This
pattern is dramatically faster than re-plotting from scratch every
iteration, and is the basis for any real-time or animated MATLAB
visualization.

## Summary

- Every plot element has a **handle**; hold onto it (`h = plot(...)`)
  rather than relying on `gca`/`gcf` once a script creates more than one
  figure or axes.
- Prefer `tiledlayout`/`nexttile` over `subplot` for new code — shared
  titles/labels and tile spanning make multi-panel figures much less
  fiddly.
- Match plot type to data shape: `scatter` for point clouds with extra
  encoded dimensions, `histogram` for distributions, `surf`/`contour` for
  functions of two variables.
- `exportgraphics` is the modern way to save publication-quality figures,
  with a clear raster/vector distinction.
- For anything animated, mutate `XData`/`YData` on an existing handle and
  call `drawnow`, rather than re-plotting every frame.
