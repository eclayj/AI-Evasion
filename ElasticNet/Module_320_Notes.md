# AI Evasion — Sparsity Attacks (Module 320)

These notes are being built section by section as a practical study guide and COAE exam reference. Formulas are read aloud before their meaning is summarized.

## Section 1 — Introduction to Sparsity Evasion Attacks

### The central change: count edits, not only their size

A sparsity attack seeks a model error by modifying as few input features as possible. For an image, a feature might be one pixel value in one color channel; for text, it might be a token or an encoded feature. The count is commonly described with the L-zero pseudo-norm:

`||x_adv - x||_0 = |{ i : (x_adv)_i != x_i }|`

1. **Say it:** “the L-zero norm of x-adv minus x equals the number of indices i for which coordinate i of x-adv is not equal to coordinate i of x.”
2. **Pieces:** `x` is the original input; `x_adv` (“x-adv”) is the adversarial input; `i` (“eye”) picks one coordinate. The braces collect indices where the two values differ. The outer vertical bars mean “count the members.”
3. **Together:** count how many coordinates changed, regardless of the magnitude of each change.

Example: original `[0.1, 0.3, 0.5, 0.7, 0.9]`, adversarial `[0.1, 0.3, 0.8, 0.7, 0.4]`. Only coordinates 3 and 5 differ, so the L-zero count is 2. Their changes have magnitudes 0.3 and 0.5, but those magnitudes do not change the count. For comparison, L1 totals magnitudes (`0.8` here); squared L2 totals their squares (`0.34` here). L0 asks “how many?”, while L1/L2 also measure “how much?”.

L0 is called a **pseudo-norm** because it counts nonzero entries but does not satisfy every mathematical rule required of a true norm (in particular, scaling a nonzero vector does not scale its L0 count). Minimizing it directly is a difficult discrete/combinatorial problem. A common strategy is to encourage sparsity indirectly with L1, whose geometry and proximal soft-threshold operation can create exact zeros.

### Threat model and input validity

The page considers an attacker at inference time: the model already exists and is producing predictions. In a **white-box** setting, the attacker can evaluate gradients through the model. In a **black-box** setting, the attacker may estimate importance with queries or use a separate surrogate model and transfer the pattern. The primary budget can be L0 (maximum number of changed coordinates); L2 or L-infinity limits may also cap change size. For image pixels represented in `[0,1]`, updates must respect those valid bounds.

If a model normalizes an input, `x_hat = (x - mu) / sigma` is read “x-hat equals x minus mu, divided by sigma.” `mu` is the centering average; `sigma` is the scale. The chain rule carries derivatives through this transform. For one coordinate, if `x_hat = (x - mu)/sigma`, then `d x_hat / d x = 1/sigma`; a loss gradient with respect to x-hat is therefore multiplied by `1/sigma` to obtain its contribution with respect to x. The model's preprocessing changes gradient scale, not the fact that gradients can guide feature selection.

### Two ways to create sparse changes

- **ElasticNet / EAD:** adds an L1 penalty to a smooth distortion and adversarial objective. L1 encourages many perturbation coordinates to become zero, giving an L0-like outcome while keeping the optimization compatible with a proximal update. It encourages sparsity; it is not a hard guarantee of an exact maximum feature count.
- **JSMA:** uses input derivatives to score features with a saliency map and changes one or a small number of selected features per iteration. A feature-count budget is explicit. It chooses features based on their effect on a target class and competing classes.

Both methods can use gradients, but the L0 constraint and feature-selection procedure differ. Later sections implement these methods; this Introduction adds no new executable code.

### Why sparse edits matter and COAE quick recall

Few-feature changes reflect settings where only a few values can be changed, and they can help explain which features influence a model's decision. Sparse does not guarantee visually imperceptible or undetectable: a single large or important change may be obvious. Defenses that measure only global noise may miss a small concentrated edit, so evaluate feature counts and individual magnitudes as separate properties.

- If the question asks “how many input dimensions changed?”, answer L0.
- If it asks about per-feature magnitude limits, think L-infinity; total absolute magnitude, L1; squared energy, squared L2.
- EAD/ElasticNet encourages L0-like sparsity through L1 and proximal thresholding; JSMA selects features under an explicit L0-style budget.
- White-box means gradient access through the target model; black-box means estimates or transfer without direct gradient access.
- Keep the threat model explicit: inference-time changes, valid input bounds, and any feature-count or magnitude limits.

## Section 2 — ElasticNet

### Why this attack is useful

An adversarial example is an input changed so that a model predicts incorrectly. Different attacks measure and limit the change in different ways. FGSM usually takes one step in the direction that most increases the loss under a fixed L-infinity bound; it is fast, but that bound limits each coordinate rather than directly asking for the fewest changed pixels. DeepFool repeatedly approximates the decision boundary and seeks a small L2 change to cross it. ElasticNet combines squared L2 and L1 costs: it seeks a successful example with small overall distortion while encouraging many individual input coordinates to remain exactly unchanged.

The attack is named for the combined penalty, not because the result must be visually imperceptible. Sparsity means that many entries of the perturbation are zero. Whether the changed entries are noticeable depends on their size, location, image, model, and evaluation method.

### Read the central formula

`D(delta) = ||delta||_2^2 + beta * ||delta||_1`

1. **Say it:** “D of delta equals the squared L-two norm of delta plus beta times the L-one norm of delta.”
2. **Pieces:** `delta` is the adversarial input minus the original input. The squared L2 term adds the square of every change. The L1 term adds the absolute size of every change. `beta` (said “BAY-tuh”) controls how much the L1 contribution matters compared with squared L2.
3. **Together:** prefer perturbations with a low combined cost; increasing beta makes the optimizer value a lower total absolute change more strongly, which tends to favor fewer changed coordinates when the other settings and success requirement allow it.

For a small example, compare two possible changes to a 28-by-28 image. Pattern A changes 100 pixels by 0.10 each. Pattern B changes 10 pixels by about 0.316 each. Their squared L2 costs are approximately `100 * 0.10^2 = 1` and `10 * 0.316^2 ≈ 1`. Their L1 costs are `100 * 0.10 = 10` and `10 * 0.316 ≈ 3.16`. If `beta = 0.1`, the combined costs are about 2.00 and 1.316, so this distance term favors the more concentrated pattern B. The full attack still must fool the model, and the final result depends on the adversarial loss, `c`, bounds, and optimization. A larger `beta` does not guarantee a particular number of altered pixels.

### The full optimization has distinct knobs

The complete objective is conceptually `c * adversarial_loss + D(delta)`, subject to valid input bounds. Read `c` as “see”: it scales pressure to achieve the attack's classification goal relative to the distance cost. A binary search adjusts `c` per input: if the attack fails, it raises the lower bound to give success more weight; after success, it lowers the upper bound to search for a less costly successful example. This is a search for a useful trade-off, not a proof that a global minimum was found.

Keep these parameters separate:

- `beta`: relative weight of L1 sparsity against squared L2 in the distance term.
- `c`: weight of the adversarial loss against the distance term.
- `kappa` (confidence): required classification margin in the adversarial loss.

FISTA divides this objective into a smooth part (classification loss plus squared L2, which can use gradients) and the nonsmooth L1 part (handled by a proximal soft-threshold step). That step can set perturbation coordinates exactly to zero. It operates on `delta`, not directly on the image, so unchanged pixels stay at their original values.

### COAE quick recall

- If a question emphasizes **few altered features/pixels**, identify L1 and its sparsity effect; in ElasticNet the proximal soft-threshold step is what can make coordinates exactly zero.
- If it emphasizes **large individual changes being costly**, identify squared L2.
- ElasticNet distance is **squared L2 + beta times L1**; do not omit the square or interchange beta with `c` or `kappa`.
- FGSM: one fast gradient sign step, commonly described with an L-infinity budget. DeepFool: repeated local boundary approximation with an L2-oriented objective. ElasticNet: adversarial objective plus L1/L2 distance trade-off, commonly optimized with FISTA.
- “Sparse” describes how many coordinates change; it does not by itself guarantee that an example is imperceptible or undetectable.
- The HTB lesson's full attack code appears later in this notebook. This theory subsection adds no separate attack implementation.

## Section 3 — Environment Setup

### Why this setup exists

ElasticNet is computationally heavier than FGSM because each FISTA iteration runs the model and computes gradients, and the iterations repeat inside a binary search over the trade-off parameter. The HTB Evasion Library supplies shared models, MNIST data loaders, training helpers, and plotting utilities so the lesson can focus on attack mechanics. The notebook installs/updates this library from the GitHub repository using `%pip install --upgrade git+https://...`; this syntax means pip fetches the package code from Git and installs it into the notebook's Python environment.

For a production or long-lived study environment, pin the library to a known release or commit and record package versions. Installing from a moving Git branch may produce different code later. That is a best-practice distinction; the HTB notebook uses its supplied installation method.

### Reproducibility and device choice

`set_reproducibility(1337)` sets random seeds for the experiment's supported libraries. Read `1337` as “one thousand three hundred thirty-seven”; it is a chosen seed value, not a special mathematical constant. Using the same seed makes random initialization and data order more comparable across runs. A seed does not guarantee exact repeatability across different GPU hardware, CUDA libraries, PyTorch versions, or nondeterministic kernels. Record those details when exact experiment reproducibility matters.

`torch.device("cuda" if torch.cuda.is_available() else "cpu")` reads “choose the CUDA device if PyTorch reports CUDA available; otherwise choose the CPU.” CUDA is NVIDIA's GPU-computing interface used by PyTorch. GPU execution helps for repeated forward/backward passes, but only if PyTorch is installed with compatible CUDA support and a compatible GPU/driver is available. Device detection in code is safer than assuming that any machine has a working GPU.

The HTB utility `use_htb_style()` applies the course's color theme to plots. This is presentation setup; it does not affect gradients, model predictions, or attack mathematics.

### Target model, data layout, and checkpoint

MNIST contains 28-by-28 grayscale digit images and ten classes (digits 0 through 9). The notebook uses `MNISTClassifierWithDropout`, with two convolutional layers and two fully connected layers. Convolutions learn local visual features; dropout randomly disables a fraction of activations during training, which discourages over-reliance on a small set of features and helps reduce overfitting. During evaluation, dropout is normally turned off so a prediction is stable.

A batch tensor is ordered `(batch, channels, height, width)`. For example, `(128, 1, 28, 28)` means 128 images, one grayscale channel per image, 28 rows, and 28 columns. The multiplication `28 * 28 = 784` is read “twenty-eight times twenty-eight equals seven hundred eighty-four”: one image has 784 pixel values. With float32 storage, each value takes 4 bytes, so `784 * 4 = 3,136` bytes per image. For 128 images, `128 * 3,136 = 401,408` bytes, about 392 KiB, for the input pixels alone. The model's intermediate activations, their gradients, and optimizer state require additional memory; HTB estimates about 50–80 MB per batch for the full training workload. Doubling batch size roughly doubles much of the per-batch working memory.

The notebook checks for `output/mnist_target.pth`. If that checkpoint exists, it loads the learned weights; otherwise it trains for five epochs and saves them. The first run therefore includes training, while subsequent runs can skip that work. A checkpoint stores the learned parameter tensors (and potentially related state, depending on the saving helper); it avoids retraining but does not by itself document the package versions or hardware used.

### Notebook code map

- Cell 1 installs the shared HTB library in the active Jupyter kernel environment.
- Cell 2 imports PyTorch and course helpers, sets seeds, applies the plotting theme, and chooses CUDA or CPU.
- Cell 5 creates MNIST loaders, selects batch size 128, checks/creates the model checkpoint, trains only when needed, and evaluates accuracy.
- `num_classes=10` configures ten model output scores, one for each digit class; the later attack code uses those scores to define adversarial loss.

### COAE quick recall

- A seed improves repeatability; it does not guarantee bit-for-bit identity across hardware and software stacks.
- `(batch, channels, height, width)` describes the image tensor axes; e.g. `(128, 1, 28, 28)` is 128 grayscale MNIST images.
- Input memory = number of values times bytes per value; training needs extra memory for activations and gradients.
- Use CUDA only when PyTorch can access compatible GPU support; fall back cleanly to CPU.
- Cache trained weights to avoid repeated training during attack-parameter experiments.
- Treat HTB's library-install command as course setup; for repeatable environments, pin versions and record the environment.

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

## Module wrap-up — One FISTA iteration by hand

This small example uses a three-coordinate **perturbation** vector so each operation is visible. It follows the HTB teaching implementation's momentum coefficient `k / (k + 3)`, not the textbook `t_k` recurrence. We minimize a simple smooth-plus-L1 objective:

`f(delta) = 1/2 * ||delta - a||_2^2` plus `beta * ||delta||_1`,

where target vector `a = [1.2, 0.4, -0.6]`, current perturbation `delta_k = [0.2, -0.1, 0]`, look-ahead point `y_k = [0.3, -0.2, 0.1]`, step size `eta = 0.5`, L1 weight `beta = 0.2`, and iteration `k = 1`. `delta_k` is the previous perturbation; `y_k` includes a small carried-forward movement from earlier steps. The example is arithmetic for learning, not an image attack run.

### 1. Gradient at the look-ahead point

For `f(delta) = 1/2 * ||delta - a||_2^2`, the gradient is `gradient f at y_k = y_k - a`. Read it as “the gradient of f at y sub k equals y sub k minus a.” The one-half cancels the factor of two that would otherwise appear when differentiating a square.

`[0.3, -0.2, 0.1] - [1.2, 0.4, -0.6] = [-0.9, -0.6, 0.7]`.

The gradient points uphill. The first two coordinates are negative because their current values are below their targets; the third is positive because its value is above its target. A descent update subtracts this gradient.

### 2. Smooth gradient step

`z = y_k - eta * gradient f at y_k`, read “z equals y sub k minus eta times the gradient of f at y sub k.” `eta` is the step size.

`z = [0.3, -0.2, 0.1] - 0.5 * [-0.9, -0.6, 0.7] = [0.75, 0.10, -0.25]`.

For example, coordinate 1 becomes `0.3 - 0.5*(-0.9) = 0.75`. Subtracting a negative moves it upward toward its target.

### 3. L1 proximal soft-threshold step

The threshold is `eta * beta = 0.5 * 0.2 = 0.1`. Soft thresholding is `S_tau(z_i) = sign(z_i) * max(|z_i| - tau, 0)`, read “S sub tau of z sub i equals the sign of z sub i times the maximum of the absolute value of z sub i minus tau, or zero.” It removes `tau` from each magnitude and preserves the sign; values whose magnitude is at most `tau` become exactly zero.

- `0.75` becomes `0.75 - 0.10 = 0.65`.
- `0.10` becomes `0`, because `max(0.10 - 0.10, 0) = 0`.
- `-0.25` becomes `-(0.25 - 0.10) = -0.15`.

Thus the new perturbation is `delta_(k+1) = [0.65, 0, -0.15]`. One coordinate is exactly zero, so that feature remains unchanged relative to the original input. In the actual image code, the threshold acts on `adv_image - original_image`, then the original image is added back and valid pixel bounds are applied.

### 4. Momentum for the next look-ahead point

The course coefficient is `m_k = k / (k + 3)`, read “m sub k equals k divided by k plus three.” At `k = 1`, `m_k = 1/4 = 0.25`. The implementation updates `y_(k+1) = delta_(k+1) + m_k * (delta_(k+1) - delta_k)`, read “y at k plus one equals the new perturbation plus the momentum coefficient times the new-minus-previous perturbation.”

The recent movement is `[0.65, 0, -0.15] - [0.2, -0.1, 0] = [0.45, 0.1, -0.15]`. Carrying one quarter of that movement gives `[0.1125, 0.025, -0.0375]`, so the next look-ahead point is `[0.7625, 0.025, -0.1875]`.

### What happened, in plain English

The gradient moved the candidate toward lower smooth loss. The proximal step then shrank all three changes and removed the small middle change completely. Momentum carried one quarter of the difference between this perturbation and the previous perturbation into the next look-ahead point. In the next iteration, the gradient will be evaluated at that look-ahead point, while the proximal step will again act on the perturbation. This is the course code's simplified FISTA teaching version; full image code also includes model loss, image bounds, batches, and per-input trade-off constants.
