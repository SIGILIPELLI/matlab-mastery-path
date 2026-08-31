# 03 · Advanced Optimization & Machine Learning Toolbox

!!! note "Verification note"
    MATLAB, Optimization Toolbox, and Statistics and Machine Learning
    Toolbox were not available in the environment used to write this
    page. Algorithm behavior and function signatures below are
    documented, version-stable toolbox features, hand-traced against
    the MathWorks documentation and cross-checked conceptually against
    equivalent scikit-learn behavior where useful, rather than executed
    in MATLAB itself.

Level 3 Module 08 covered `fminsearch`/`fmincon` for general nonlinear
optimization. This module goes further into constrained/multi-objective
optimization and MATLAB's machine learning toolbox — the classification,
regression, and clustering functions that turn MATLAB into a full data
science environment, not just a numerical computing one.

## Global vs. local optimization

`fmincon` and `fminunc` are **local** optimizers — they converge to
whichever minimum is nearest the starting point, which may not be the
global minimum for a non-convex objective:

```matlab
% multi-modal function with several local minima
f = @(x) sin(3*x) + 0.1*x.^2;

x1 = fminsearch(f, 0);     % converges to a minimum near x=0
x2 = fminsearch(f, 5);     % may converge to a DIFFERENT minimum, depending on start
```

`GlobalSearch` and `MultiStart` (Global Optimization Toolbox) address
this by trying many starting points systematically and keeping the best
result found:

```matlab
opts = optimoptions('fmincon', 'Display', 'off');
problem = createOptimProblem('fmincon', 'objective', f, 'x0', 0, 'options', opts);
gs = GlobalSearch;
[xBest, fBest] = run(gs, problem);
```

`ga` (genetic algorithm) and `particleswarm` are population-based
metaheuristics — useful when the objective is non-differentiable,
discontinuous, or has many local minima where gradient-based local
search struggles:

```matlab
xBest = ga(f, 1, [], [], [], [], -10, 10);   % 1 variable, bounded to [-10, 10]
```

## Multi-objective optimization

```matlab
% minimize both cost and weight simultaneously — generally in tension
objectives = @(x) [x(1)^2 + x(2), (x(1) - 2)^2 + x(2)^2];
[x, fval] = gamultiobj(objectives, 2, [], [], [], [], [0 0], [5 5]);
```

`gamultiobj` returns a **Pareto front** — a set of solutions where no
objective can be improved without worsening another — rather than a
single "best" answer, since with genuinely competing objectives there
often is no single optimum, only tradeoffs the decision-maker chooses
among afterward.

## Machine learning: classification

```matlab
load fisheriris    % classic built-in dataset: 150 iris flowers, 4 measurements, 3 species
X = meas;                    % 150x4 predictor matrix
Y = species;                 % 150x1 cell array of class labels

cv = cvpartition(Y, 'HoldOut', 0.3);   % 70/30 train/test split, stratified by class
XTrain = X(training(cv), :);
YTrain = Y(training(cv), :);
XTest  = X(test(cv), :);
YTest  = Y(test(cv), :);

model = fitcsvm(XTrain, YTrain, ...
    'KernelFunction', 'rbf', ...
    'Standardize', true);
```

`fitcsvm` trains a binary support vector machine; for the 3-class iris
problem, `fitcecoc` (error-correcting output codes) combines several
binary SVMs to handle multi-class classification:

```matlab
model = fitcecoc(XTrain, YTrain);
predictions = predict(model, XTest);

accuracy = sum(strcmp(predictions, YTest)) / numel(YTest);
disp(accuracy);   % well-separated classes like iris typically yield high accuracy, often >0.9
```

`'Standardize', true` is important for SVMs specifically — since the
RBF kernel depends on Euclidean distance between feature vectors,
un-normalized features with very different scales (e.g. one column in
millimeters, another in kilometers) would let the larger-scale feature
dominate distance calculations regardless of its actual predictive
value.

### Other classifiers, same interface pattern

```matlab
treeModel = fitctree(XTrain, YTrain);          % decision tree
forestModel = fitcensemble(XTrain, YTrain);    % ensemble (boosting/bagging)
knnModel = fitcknn(XTrain, YTrain, 'NumNeighbors', 5);
nbModel = fitcnb(XTrain, YTrain);              % naive Bayes
```

Every `fitc*` function shares the `predict(model, XNew)` interface —
switching classifiers to compare performance is a one-line change, not
a rewrite of the surrounding evaluation code.

```matlab
confMat = confusionmat(YTest, predictions);
confusionchart(YTest, predictions);   % visual confusion matrix
```

## Regression

```matlab
load carsmall
X = [Weight, Horsepower];
Y = MPG;

validRows = ~any(isnan(X), 2) & ~isnan(Y);
X = X(validRows, :);
Y = Y(validRows);

regModel = fitrensemble(X, Y, 'Method', 'LSBoost');
predictedMPG = predict(regModel, X);

residuals = Y - predictedMPG;
rmse = sqrt(mean(residuals.^2));
disp(rmse);
```

`fitlm` (linear model, closer to classical statistics than machine
learning) provides interpretable coefficients and p-values when the
relationship is expected to be roughly linear:

```matlab
lm = fitlm(X, Y, 'VarNames', {'Weight', 'Horsepower', 'MPG'});
disp(lm);           % coefficient table with p-values, R^2
plot(lm);           % diagnostic residual plots
```

## Unsupervised learning: clustering

```matlab
X = [randn(50,2)*0.5 + [1 1]; randn(50,2)*0.5 + [5 5]; randn(50,2)*0.5 + [1 5]];

[idx, centroids] = kmeans(X, 3);   % k-means with k=3 known clusters

gscatter(X(:,1), X(:,2), idx);
hold on;
plot(centroids(:,1), centroids(:,2), 'kx', 'MarkerSize', 15, 'LineWidth', 3);
```

`kmeans` requires the number of clusters `k` as an input; when `k` is
unknown, the **elbow method** (plotting within-cluster sum of squares
against increasing `k` and looking for where the improvement rate drops
off) or `evalclusters` gives a data-driven estimate:

```matlab
eval = evalclusters(X, 'kmeans', 'silhouette', 'KList', 1:8);
disp(eval.OptimalK);
```

`clusterdata` and `linkage`/`dendrogram` support hierarchical clustering
when a cluster-count hierarchy (not just a flat partition) is useful —
common for taxonomic or nested-group data.

## Cross-validation and avoiding overfitting

```matlab
cvModel = fitcsvm(X, Y, 'KFold', 5);       % 5-fold cross-validation, no manual split needed
cvLoss = kfoldLoss(cvModel);               % average misclassification rate across folds
```

`'KFold', 5` internally partitions the data 5 ways, trains 5 models
(each leaving out one fold as validation), and `kfoldLoss` averages
their held-out error — a more reliable performance estimate than a
single train/test split, since it isn't sensitive to which particular
rows happened to land in the test set.

```matlab
% hyperparameter tuning with cross-validation baked in
model = fitcsvm(X, Y, 'OptimizeHyperparameters', 'auto', ...
    'HyperparameterOptimizationOptions', struct('KFold', 5, 'ShowPlots', false));
```

`'OptimizeHyperparameters', 'auto'` runs Bayesian optimization over the
classifier's key hyperparameters (e.g. `BoxConstraint` and
`KernelScale` for an SVM), evaluating each candidate via cross-validated
loss rather than a single hold-out split, avoiding hyperparameters
overfit to one lucky test partition.

## Choosing an approach

| Situation | Tool |
|---|---|
| Non-convex objective, need best-effort global minimum | `GlobalSearch`/`MultiStart`, `ga`, `particleswarm` |
| Genuinely competing objectives | `gamultiobj` (Pareto front) |
| Predict a category | `fitcsvm`, `fitctree`, `fitcensemble`, `fitcknn` |
| Predict a continuous value | `fitrensemble`, `fitlm`, `fitrsvm` |
| Find structure with no labels | `kmeans`, hierarchical clustering |
| Reliable performance estimate | `'KFold'` cross-validation, not a single split |
| Tune model settings without manual guessing | `'OptimizeHyperparameters', 'auto'` |

## Practice

1. Using the iris dataset structure above, train both `fitcecoc` and
   `fitctree` models on the same train/test split and compare their
   test-set accuracy and confusion matrices; explain a plausible reason
   one might outperform the other on this particular dataset.
2. Explain, with a concrete pair of features at very different physical
   scales, why `'Standardize', true` changes an RBF-kernel SVM's
   decision boundary, tracing through how Euclidean distance is
   affected.
3. Using `kmeans` with `k` swept from 1 to 8 on a synthetic dataset with
   3 true clusters, describe what the within-cluster sum-of-squares
   curve should look like and where you'd expect the "elbow."
4. Explain why a single 70/30 train/test split can give a misleadingly
   optimistic or pessimistic accuracy estimate purely by chance, and how
   5-fold cross-validation's averaging addresses that; connect this to
   why hyperparameter search should be wrapped in cross-validation
   rather than tuned against one fixed test set.
