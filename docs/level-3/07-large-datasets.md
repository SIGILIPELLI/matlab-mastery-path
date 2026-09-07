# 07 · Working with Large Datasets

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. `datastore`, `tall`, and memory-mapped file behaviors are
    documented, version-stable MATLAB features, hand-traced against the
    MATLAB documentation rather than executed in MATLAB itself.

Level 2's tables and `readtable` assume a dataset fits comfortably in
RAM. Real datasets — logs spanning years, sensor data at high sample
rates, multi-gigabyte CSVs — often don't. This module covers MATLAB's
tools for data that's too large to load at once: `datastore` for
chunked/streaming access, `tall` arrays for out-of-memory computation
with familiar syntax, and practical memory-management techniques.

## The core problem

```matlab
T = readtable('sensor_log_50gb.csv');  % attempts to load the whole file into RAM — fails or thrashes
```

`readtable` reads everything into memory as one in-memory table. When
the file is larger than available RAM, this either errors (`Out of
memory`) or triggers swapping so severe the process becomes unusable.
The fix is to never materialize the whole dataset at once — process it
in pieces.

## `datastore`: chunked access to out-of-memory data

```matlab
ds = datastore('sensor_log_50gb.csv', 'TreatAsMissing', 'NA');
ds.SelectedVariableNames = {'Timestamp', 'Temperature', 'Pressure'};

reset(ds);
total = 0;
count = 0;
while hasdata(ds)
    chunk = read(ds);          % reads one manageable chunk, e.g. a few thousand rows
    total = total + sum(chunk.Temperature, 'omitnan');
    count = count + height(chunk);
end
meanTemp = total / count;
```

`datastore` auto-detects the file type (CSV, spreadsheet, images,
key-value, custom) and exposes a uniform `read`/`hasdata`/`reset`
interface regardless of format, reading fixed-size chunks so memory
usage stays bounded no matter how large the underlying file is.

```matlab
imds = imageDatastore('training_images/', 'IncludeSubfolders', true, ...
                       'LabelSource', 'foldernames');
% imds.Labels auto-derived from subfolder names — common ML data-loading pattern
while hasdata(imds)
    img = read(imds);      % one image at a time
end
```

### `tabularTextDatastore` for large delimited files specifically

```matlab
tds = tabularTextDatastore('logs_*.csv', 'FileExtensions', '.csv');
tds.ReadSize = 10000;    % rows per chunk, tune for memory/throughput tradeoff
tds.VariableTypes{3} = 'double';   % force a column's type if auto-detection guesses wrong
```

A wildcard pattern (`logs_*.csv`) treats matching files as one logical
dataset, chunking across file boundaries transparently — useful for
data split across daily/hourly log files.

## `tall` arrays: familiar syntax, deferred execution

`tall` wraps a `datastore` in an array-like interface so you can write
ordinary-looking MATLAB syntax (`mean`, `sum`, indexing, arithmetic)
against data that never fully loads into memory:

```matlab
ds = datastore('sensor_log_50gb.csv');
tt = tall(ds);              % tall table backed by the datastore
tt.Temperature = fillmissing(tt.Temperature, 'linear');

meanTemp = mean(tt.Temperature, 'omitnan');   % NOT computed yet — this builds a plan
result = gather(meanTemp);                    % NOW MATLAB executes the plan, chunk by chunk
```

This is **deferred (lazy) evaluation**: operations on a `tall` array
build up a computation graph rather than running immediately. Nothing
actually executes until `gather` is called, at which point MATLAB
optimizes the whole chain (fusing operations, minimizing passes over the
data) and streams through the underlying datastore in chunks,
accumulating the result.

```matlab
tt = tall(ds);
highTemp = tt(tt.Temperature > 100, :);   % filter — still lazy
grouped = groupsummary(highTemp, 'SensorID', 'mean', 'Pressure');  % still lazy
finalResult = gather(grouped);   % single pass triggers all of the above
```

Chaining several `tall` operations before one `gather` lets MATLAB fuse
them into a single streaming pass, rather than reading the file once per
operation — significantly more efficient than calling `gather` after
each intermediate step.

### What works on `tall` arrays

Most common functions have `tall`-aware overloads: `sum`, `mean`, `std`,
`min`, `max`, `sort`, `unique`, `histogram`, logical indexing,
`groupsummary`, string functions, and more. Functions without a
built-in `tall` overload can sometimes be applied per-chunk via
`cellfun`-like patterns, but not all algorithms translate to a
single-pass or bounded-memory form (e.g., an algorithm needing full
random access to sorted data may need `sortrows` support, which
`tall` does provide, but with a cost — sorting out-of-memory data
requires spooling to disk).

```matlab
summary(tt)              % tall-aware summary statistics
[C, ia, ic] = unique(tt.SensorID);   % tall-aware unique
```

## Memory-mapped files: `memmapfile`

For large *binary* files with a known fixed record layout, `memmapfile`
maps the file directly into the address space without reading it into
RAM up front — the OS pages sections in on demand:

```matlab
m = memmapfile('readings.bin', ...
    'Format', {'double', [1 1], 'timestamp'; ...
               'single', [1 3], 'xyz'; ...
               'uint8',  [1 1], 'flag'});

record500 = m.Data(500);       % reads only the bytes for record 500, not the whole file
allTimestamps = [m.Data.timestamp];   % still touches every record, but page-by-page
```

This is appropriate when records have a fixed, known binary layout
(sensor firmware logs, custom binary formats) — for delimited text or
irregular structure, `datastore` is the right tool instead.

## Reducing memory footprint of in-memory data

When data does fit but is inefficiently stored, several tactics matter:

```matlab
x = int32(sensorReadings);        % 4 bytes/elem instead of double's 8, if precision allows
x = single(sensorReadings);       % 4-byte float instead of 8-byte double

s = sparse(mostlyZeroMatrix);     % stores only non-zeros — huge savings for sparse data
whos                              % inspect variable memory footprint in the workspace

categoricalCol = categorical(stringColumn);   % dedups repeated strings into small integer codes
```

`categorical` in particular can shrink a column of a few thousand
distinct repeated strings by an order of magnitude compared to storing
each row as its own string, while still supporting comparisons,
`unique`, and grouping naturally.

```matlab
clear largeIntermediate;   % explicitly free a variable once done with it
```

## Choosing the right tool

| Situation | Tool |
|---|---|
| Delimited/tabular file bigger than RAM, need full-dataset stats | `tall` over a `datastore` |
| Need to scan/process chunk-by-chunk with custom logic per chunk | `datastore` directly (`read`/`hasdata`) |
| Fixed-layout binary file, need random access to specific records | `memmapfile` |
| Data fits in RAM but wastes space | narrower numeric types, `sparse`, `categorical` |
| Folder of images/files for ML training | `imageDatastore` / `fileDatastore` |

## How It Actually Works

MATLAB's ordinary numeric arrays require their **entire contents to fit
in contiguous RAM** — for a dataset larger than physical memory, this
isn't a performance problem to tune around, it's a hard allocation
failure. `datastore`/`tall` arrays exist specifically to route around
this: a `tall` array doesn't materialize your full dataset in memory at
all — it builds a **lazy execution plan** (a directed graph of the
operations you've chained: filter, then group, then aggregate) and only
touches actual data when you call `gather`, at which point MATLAB
processes the underlying files in **chunks** small enough to fit in
memory, applying the whole operation chain to each chunk and combining
partial results — conceptually the same chunked, out-of-core
processing strategy used by tools like Dask or Spark, just presented
through ordinary-looking MATLAB array syntax.

This lazy-evaluation model changes debugging mechanics: printing a `tall`
array mid-pipeline doesn't show you data, it shows you the *unevaluated
plan*, because no computation has actually run yet — this is often
surprising to people used to MATLAB's normally eager, immediate-execution
semantics (Module 01's REPL model), and it's why `tall` workflows
explicitly separate "build up the operation chain" from "call `gather` to
force execution," rather than executing each line as it's typed.

Memory-mapped access (`memmapfile`) takes a different approach entirely:
rather than chunking through a dataset via a lazy pipeline, it maps a
file's bytes directly into the process's virtual address space, letting
the operating system's page cache — not MATLAB — decide which parts of
the file are actually resident in physical RAM at any moment, paging
sections in and out on access.

*Note: based on MathWorks' documented `tall`/`datastore` lazy-evaluation
and `memmapfile` architecture; not executed against a real large dataset
in this environment.*

## Practice

1. Given a hypothetical 20 GB CSV of `Timestamp, SensorID, Value` rows,
   write a `datastore`-based loop that computes the per-`SensorID`
   running mean of `Value` without ever loading the full file.
2. Rewrite the same computation using `tall` + `groupsummary` +
   `gather`, and explain (in terms of passes over the data) why chaining
   the filter and the groupsummary before a single `gather` is more
   efficient than gathering after each step.
3. Given a binary sensor log with a fixed 13-byte record (`double`
   timestamp + `single` value + `uint8` flag), write the `memmapfile`
   `Format` specification and a snippet extracting only records where
   `flag == 1` without loading the whole file into a MATLAB array first.
4. A table column has 2 million rows but only 5 distinct string values.
   Explain, with an approximate memory estimate, why converting it to
   `categorical` matters, and what operations remain fully supported
   after the conversion.
