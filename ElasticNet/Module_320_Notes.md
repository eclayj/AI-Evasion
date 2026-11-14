# AI Evasion — Sparsity Attacks (Module 320)

These notes are being built section by section as a practical study guide and COAE exam reference. Formulas are read aloud before their meaning is summarized.

## Section 4 — Proximal Operators and FISTA Framework

### Why L1 needs a different optimization step

The L1 penalty on one scalar is `|x|`, read “absolute value of x.” Its slope is `+1` to the right of zero and `-1` to the left, but there is no single derivative at zero because the graph has a sharp corner. Ordinary gradient descent can make values tiny, but it has no ordinary gradient rule at that corner to snap them to zero. That often leaves pseudo-sparsity—many tiny values—instead of true sparsity—many exact zeros.

FISTA splits the objective into a smooth part and a nonsmooth part. The smooth part can use backpropagation and gradients. The nonsmooth L1 penalty is handled by a proximal operator. Nesterov-style momentum helps the repeated updates converge faster.

### Proximal operator

`prox_(lambda h)(z) = arg min_x { 1/2 ||x - z||_2^2 + lambda h(x) }`

**Read aloud:** “The proximal operator for lambda h, applied to z, equals the value of x that minimizes one half the squared L2 distance between x and z, plus lambda times h of x.”

- `z` is the candidate point before the adjustment.
- `||x - z||_2^2` (“the squared L2 norm of x minus z”) measures squared Euclidean distance. The one-half is a convenient scaling.
- `h(x)` is the penalty we want the answer to satisfy.
- `lambda` sets the strength of that penalty.
- `arg min` means “the argument value that minimizes.” It asks which `x` gives the lowest expression, not what that lowest score is.

**Whole formula in English:** choose an output that stays near candidate `z` while also reducing the penalty. It generalizes projection: if the penalty is zero inside an allowed set and infinite outside, minimizing this expression selects the closest valid point. PGD’s projection is a familiar constrained-optimization example.

### L1 proximal operator: soft thresholding

For each coordinate `i`, the soft-threshold operator is:

`S_lambda(z_i) = z_i - lambda` when `z_i > lambda`; `0` when `|z_i| <= lambda`; and `z_i + lambda` when `z_i < -lambda`.

**Read aloud:** “S sub lambda of z sub i.” `z_i` means coordinate `i` of vector `z`; `lambda` is the threshold. The absolute value condition means the coordinate is within `lambda` of zero.

**Three cases:**

1. Above the positive threshold: subtract the threshold, moving toward zero.
2. Inside the threshold band: set exactly to zero.
3. Below the negative threshold: add the threshold, again moving toward zero.

With `lambda = 0.10`, vector `[0.12, 0.08, -0.25]` becomes `[0.02, 0, -0.15]`. For the first coordinate, `0.12 - 0.10 = 0.02`; the second magnitude `0.08` is at most `0.10`, so it becomes zero; the negative third coordinate adds `0.10`, becoming `-0.15`. Larger thresholds create more zeros and shrink surviving coordinates more. Hard thresholding would discard small values without shrinking surviving ones; the L1 proximal operator is soft thresholding.

A one-coordinate view shows why the rule is a minimizer: minimize `1/2 (x_i - z_i)^2 + lambda |x_i|`. The quadratic pulls `x_i` toward `z_i`; the absolute-value penalty pulls it toward zero. For positive `x_i`, differentiating gives `x_i - z_i + lambda = 0`, so `x_i = z_i - lambda`. For negative `x_i`, the derivative gives `x_i - z_i - lambda = 0`, so `x_i = z_i + lambda`. If `|z_i| <= lambda`, zero has the lowest objective. The case split is a way of doing the one-dimensional minimization despite the nondifferentiable corner at zero. Since both L1 and squared L2 separate into sums over coordinates, this operation can be applied independently to each component.

### Proximal gradient and FISTA

For objective `f(x) + h(x)`, where `f` is smooth and `h` may be nonsmooth, proximal gradient (ISTA) is:

`x^(k+1) = prox_(eta h)(x^k - eta * gradient f at x^k)`

**Read aloud:** “x at iteration k plus one equals the proximal operator for eta h, applied to x at iteration k minus eta times the gradient of f at x at iteration k.” Here `k` is the iteration counter, `eta` is the learning rate, and “gradient of f at x” is the vector of slopes of `f` with respect to the coordinates of `x`. The gradient step moves downhill for the smooth part; the proximal step deals with the nonsmooth term.

FISTA uses an extrapolated point and momentum:

`x^(k+1) = prox_(eta h)(y^k - eta * gradient f at y^k)`

`t_(k+1) = (1 + sqrt(1 + 4 t_k^2)) / 2`

`y^(k+1) = x^(k+1) + ((t_k - 1) / t_(k+1)) * (x^(k+1) - x^k)`

**Read aloud:** `y sub k` is “y at iteration k”; `t sub k` is “t sub k”; `x^(k+1) - x^k` is “x at the next iteration minus x at the current iteration.” That difference carries the most recent direction and distance into a new look-ahead point. Standard FISTA has an `O(1/k^2)` objective convergence rate under its assumptions, compared with `O(1/k)` for basic proximal gradient. `O` (“big O”) describes how the error scales as iteration count grows; the square in the denominator means error decreases faster asymptotically.

HTB also presents the teaching approximation `k / (k + 3)` for a practical momentum coefficient. Read it “k divided by k plus three.” It grows from a small value toward one. It approximates FISTA-like momentum behavior but is not the exact standard `t_k` recurrence, so keep the formula and variant distinct when describing the code.

### Splitting ElasticNet's objective

With original image `x`, adversarial image `x-prime`, and perturbation `delta = x-prime - x`, ElasticNet separates:

- smooth `f`: `c` times the margin-based adversarial loss plus squared L2 distance;
- nonsmooth `h`: `beta * ||x-prime - x||_1`, read “beta times the L1 norm of x-prime minus x.”

The model loss and squared L2 term can use gradients; the L1 term is handled by soft thresholding. The proximal formula in image coordinates is `x + S_(eta beta)(z - x)`: subtract original image `x` to isolate candidate perturbation, soft-threshold that perturbation with threshold `eta * beta`, then add `x` back. For example, candidate delta `[0.12, -0.07]` with threshold `0.10` becomes `[0.02, 0]`. This creates a sparse change while retaining the original background pixels. Applying the operation directly to image values could zero pixels themselves, which is not what the attack optimizes.

### Connections and COAE recall

- FGSM: one sign-gradient step under an L-infinity budget.
- PGD: repeated gradient steps with projection into a constraint set.
- DeepFool: repeated local boundary approximation and geometric projection.
- ElasticNet/FISTA: repeated optimization balancing adversarial success, squared L2 distortion, and L1 sparsity.
- Gradient → smooth part; proximal operator → nonsmooth part; L1 proximal operator → soft thresholding; momentum → acceleration.
- `eta` controls the smooth gradient step, `beta` weights L1 sparsity, and `c` balances adversarial success against distance. They are different knobs.
- Preserve `delta = x-prime - x`; sparsity acts on delta, not on the adversarial image.

### Notebook mapping

The existing `ElasticNet/ElasticNet.ipynb` already contains the later momentum, soft-threshold, and FISTA functions. This subsection introduces their mathematical rationale but adds no new executable code. The notebook now includes a concise framework primer before its distance-metric implementation; no later-section function was rewritten.

## Section 5 — Distance Metrics

### Inputs and perturbation

Let `x` be an original image and `x-prime` its adversarial version. Their difference is `delta = x-prime - x`, pronounced “delta equals x-prime minus x.” Each coordinate of delta is the change to one pixel/channel location. A zero coordinate means that location was unchanged.

### L0, L1, and squared L2

- `||delta||_0` (“the L-zero norm of delta”) counts nonzero coordinates: how many values changed. It does not care whether each change was tiny or large.
- `||delta||_1 = sum_i |delta_i|` (“the L-one norm of delta equals the sum over i of the absolute value of delta sub i”) totals change magnitudes.
- `||delta||_2^2 = sum_i delta_i^2` (“the squared L-two norm of delta equals the sum over i of delta sub i squared”) totals squared changes. A change of `0.30` contributes `0.09`; a change of `0.15` contributes `0.0225`, one quarter as much.

Example: pattern A has 100 coordinates changed by `0.15`; pattern B has 50 changed by `0.30`. Both have L1 value `15`. Their squared-L2 values are `100 * 0.15^2 = 2.25` and `50 * 0.30^2 = 4.5`. Thus for the same total absolute change, the concentrated/larger changes have twice the squared-L2 cost. This is a numerical illustration of the quadratic penalty. It does not mean squared L2 is a perceptual metric.

ElasticNet's distance term is `D(delta) = ||delta||_2^2 + beta * ||delta||_1`, read “the squared L-two norm of delta plus beta times the L-one norm of delta.” The first part discourages concentrated large changes. The second part rewards sparsity through an L1 contribution, scaled by `beta`. With `beta = 0.1`, the two example costs become `2.25 + 0.1 * 15 = 3.75` and `4.5 + 0.1 * 15 = 6.0`. The smaller combined score prefers pattern A at these settings. `beta` does not set the adversarial-confidence margin (`kappa`) or the overall adversarial-vs-distance constant (`c`).

A nuance for exam answers: minimizing `||delta||_2` alone and minimizing its square alone have the same ordering because squaring is monotonic for nonnegative values. But once this distance is added to other loss terms with weights, replacing the unsquared norm by the squared norm can change the relative trade-off and therefore the optimizer's result. HTB's code computes the **squared** L2 term.

### Reading the PyTorch reductions

Images are batched in `(batch, channels, height, width)` order. For example, `(20, 1, 28, 28)` means 20 grayscale images, each with 1 channel and 28 by 28 pixels. The PyTorch expression `adv_images - original_images` forms delta for every image. `torch.abs(...)` takes coordinate magnitudes; `torch.sum(..., dim=(1, 2, 3))` adds channel, row, and column dimensions while preserving dimension 0. Therefore it returns one distance per image, with shape `(20,)`. Squaring delta first and reducing the same axes yields one squared-L2 distance per image. Finally, `l2_dist + beta * l1_dist` computes the combined metric independently for every batch item.

The helper measures distances; it does not itself clip values, project into a constraint region, apply proximal thresholding, or decide whether the model changed classes. Those are separate operations.

### COAE quick recall

- L0: count changed coordinates. L1: sum their absolute magnitudes. Squared L2: sum their squared magnitudes.
- ElasticNet distance: squared L2 plus `beta` times L1.
- In batched images, reduce dimensions `(1, 2, 3)` to get one value per image and preserve the batch axis.
- Check whether a formula says L2 or squared L2; HTB implements `sum(delta ** 2)`.
- `beta` scales the L1 contribution; `c` balances the adversarial loss against distance; `kappa` sets the classification margin.
- A distance function reports a score; it is not a projection, clip, or attack-success test.

### Notebook mapping

The existing `compute_distances(adv_images, original_images, beta)` implements the three sums directly. Comments now identify `delta`, each reduction formula, and the ElasticNet combination. Section 4 already introduced the FISTA machinery; this section supplies the distance measurements it optimizes and reports.
