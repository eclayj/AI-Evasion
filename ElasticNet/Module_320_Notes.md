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
