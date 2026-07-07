# Parametric Curve Parameter Estimation

This project recovers the hidden values **`theta`**, **`M`**, and **`X`** from the curve points in `xy_data.csv`.

The assignment gives only the **`x, y` points**. It does not directly give the hidden parameter `t` for each point. The main idea is to use the curve structure, undo the rotation and translation, and then fit the unknown values.

---

## Candidate Information

| Field | Details |
|------|---------|
| **Name** | Adithya Varma Kusampudi |
| **Course** | B.Tech Computer Science and Engineering (Artificial Intelligence) |
| **Roll No** | AM.SC.U4AIE23008 |
| **College** | Amrita Vishwa Vidyapeetham, Amritapuri |
| **Assignment** | Research & Development |

---

## The Given Curve

The parametric curve is:

$$
x = t\cos(\theta) - e^{M|t|}\sin(0.3t)\sin(\theta) + X
$$

$$
y = 42 + t\sin(\theta) + e^{M|t|}\sin(0.3t)\cos(\theta)
$$

The allowed parameter ranges are:

- $0^\circ < \theta < 50^\circ$
- $-0.05 < M < 0.05$
- $0 < X < 100$
- $6 < t < 60$

The score is based on the **L1 distance** between sampled points on the expected and predicted curve, so this is a curve fitting and parameter estimation problem.

---

## Main Observation

The curve has a hidden rotation structure.

First we define:

$$
u = x - X,\qquad v = y - 42
$$

Then the equations become:

$$
u = t\cos(\theta) - e^{M|t|}\sin(0.3t)\sin(\theta)
$$

$$
v = t\sin(\theta) + e^{M|t|}\sin(0.3t)\cos(\theta)
$$

Now we define:

$$
s = e^{M|t|}\sin(0.3t)
$$

Then:

$$
u = t\cos(\theta) - s\sin(\theta)
$$

$$
v = t\sin(\theta) + s\cos(\theta)
$$

This is exactly a 2D rotation:

$$
(u, v) = (t\cos\theta - s\sin\theta,\; t\sin\theta + s\cos\theta)
$$

So the original curve is a simpler curve rotated by $\theta$ and translated by $(X, 42)$.

---

## Undoing the Rotation

Because rotation matrices are easy to invert, we can recover the hidden coordinates from any guessed $\theta$ and $X$:

$$
(t, s) = (u\cos\theta + v\sin\theta,\; -u\sin\theta + v\cos\theta)
$$

This gives:

$$
t = u\cos\theta + v\sin\theta
$$

$$
s = -u\sin\theta + v\cos\theta
$$

Substituting $u = x - X$ and $v = y - 42$:

$$
t = (x - X)\cos\theta + (y - 42)\sin\theta
$$

$$
s = -(x - X)\sin\theta + (y - 42)\cos\theta
$$

For the correct parameters, this recovered value of $s$ must match:

$$
s = e^{M|t|}\sin(0.3t)
$$

So the fitting residual used in the notebook is:

$$
r = -(x - X)\sin\theta + (y - 42)\cos\theta - e^{M|t|}\sin(0.3t)
$$

where:

$$
t = (x - X)\cos\theta + (y - 42)\sin\theta
$$

---

## What `t` Means

`t` is not one constant for the whole dataset. It is the parameter that moves along the curve.

Each point has its own hidden value:

$$
t_1,\ t_2,\ t_3,\ \dots,\ t_n
$$

So:

- one row of the CSV = one point on the curve
- one point on the curve = one hidden parameter value $t_i$

The notebook does not assume the rows are ordered by `t`. This is important because the CSV points are shuffled.

---

## Notebook Process

The notebook [curve_parameter_estimation.ipynb](curve_parameter_estimation.ipynb) follows this process:

1. Load the `(x, y)` points from `xy_data.csv`.
2. Use the inverse rotation equations to recover candidate `(t, s)` values.
3. Measure how far each recovered point is from $s = e^{M|t|}\sin(0.3t)$.
4. Use Differential Evolution for a global search over `theta`, `M`, and `X`.
5. Refine the result using local least-squares optimization.
6. Apply a final small optimization directly on the total L1 score.
7. Print the final parameters.

---

## Detailed Code Process

The notebook is organized so that the explanation appears in Markdown first and the matching code cell appears directly below it.

### 1. Importing Libraries and Setting Bounds

The notebook imports `numpy`, `pandas`, and optimization functions from `scipy.optimize`.

The parameter ranges are stored as bounds:

| Parameter | Bound Used in Code |
|-----------|--------------------|
| `theta` | $0^\circ < \theta < 50^\circ$ |
| `M` | $-0.05 < M < 0.05$ |
| `X` | $0 < X < 100$ |
| `t` | $6 < t < 60$ |

Since NumPy trigonometric functions use radians, the code converts the angle bounds from degrees to radians.

### 2. Loading the Dataset

The function `load_points()` reads `xy_data.csv` using `pandas`.

The CSV has only two columns:

- `x`
- `y`

The data is converted into a NumPy array so that all points can be processed at once using vectorized operations.

### 3. Recovering Hidden Coordinates

The function `invert_transform()` applies the inverse rotation and translation.

For each point $(x_i, y_i)$, it computes:

$$
t_i = (x_i - X)\cos\theta + (y_i - 42)\sin\theta
$$

and:

$$
s_i = -(x_i - X)\sin\theta + (y_i - 42)\cos\theta
$$

This is the key step because it lets the code estimate each point's hidden `t` value without needing the CSV rows to be sorted.

### 4. Measuring the Curve Residual

The function `implicit_residuals()` checks whether each recovered point satisfies the original curve structure.

For the correct parameters:

$$
s_i = e^{M|t_i|}\sin(0.3t_i)
$$

So the residual is:

$$
r_i = s_i - e^{M|t_i|}\sin(0.3t_i)
$$

Smaller residuals mean the guessed values of `theta`, `M`, and `X` fit the curve better.

### 5. Global Search

The function `differential_evolution()` performs a global search over the allowed parameter ranges.

This is useful because the curve contains a sinusoidal term, so a simple local optimizer can get stuck in a poor local solution if the starting point is bad.

### 6. Local Refinement

After the global search finds a good region, `least_squares()` refines the parameters by minimizing the implicit residuals more accurately.

This gives a strong geometric fit to the point cloud.

### 7. Direct L1 Polishing

The assignment score is based on L1 distance, so the notebook performs one final narrow optimization using:

$$
E_{\text{L1}} =
\sum_{i=1}^{n}
\left(
|\hat{x}_i - x_i| + |\hat{y}_i - y_i|
\right)
$$

This final step slightly reduces the score compared with using only the implicit residual.

### 8. Printing the Result

The final cell prints:

- the fitted value of `theta` in radians
- the fitted value of `theta` in degrees
- the fitted value of `M`
- the fitted value of `X`
- the total L1 error
- the final equation in submission format

---

## Final Result

The clean interpretable parameters are:

$$
\theta = \frac{\pi}{6},\qquad M = 0.03,\qquad X = 55
$$

The polished numeric result from the notebook is:

$$
\theta = 0.5235983044,\qquad M = 0.0299999971,\qquad X = 54.9999983399
$$

The final equation is:

$$
\left(
t\cos(0.5235983044) - e^{0.0299999971|t|}\sin(0.3t)\sin(0.5235983044) + 54.9999983399,\;
42 + t\sin(0.5235983044) + e^{0.0299999971|t|}\sin(0.3t)\cos(0.5235983044)
\right)
$$

The notebook reports:

$$
\text{total L1} = 0.005242654119094
$$

---

## Error Function Used

For each observed point $(x_i, y_i)$, the notebook first estimates the hidden curve parameter using the inverse rotation:

$$
t_i = (x_i - X)\cos\theta + (y_i - 42)\sin\theta
$$

Then it estimates the corresponding vertical value in the unrotated coordinate system:

$$
s_i = -(x_i - X)\sin\theta + (y_i - 42)\cos\theta
$$

For the correct parameters, this value should satisfy:

$$
s_i = e^{M|t_i|}\sin(0.3t_i)
$$

So the main fitting error for each point is:

$$
r_i = s_i - e^{M|t_i|}\sin(0.3t_i)
$$

The optimizer first minimizes the squared implicit error:

$$
E_{\text{implicit}} = \frac{1}{n}\sum_{i=1}^{n} r_i^2
$$

After that, the notebook directly polishes the result using the L1 reconstruction error:

$$
E_{\text{L1}} =
\sum_{i=1}^{n}
\left(
|\hat{x}_i - x_i| + |\hat{y}_i - y_i|
\right)
$$

where $(\hat{x}_i, \hat{y}_i)$ are the reconstructed points from the fitted curve.

---

## Observations

- The CSV rows are shuffled, so assuming row order corresponds to increasing `t` gives a much worse fit.
- The inverse rotation method avoids needing to know the original row order.
- The fitted values are extremely close to the simple exact-looking values $\theta = \pi/6$, $M = 0.03$, and $X = 55$.
- The small remaining error is expected because the CSV values are rounded decimal numbers.
- The final L1 polish slightly improves the score compared with only minimizing the implicit residual.
- The recovered hidden `t` values stay inside the required range $6 < t < 60$.

---

## Project Files

- Dataset: `xy_data.csv`
- Notebook Solution: `curve_parameter_estimation.ipynb`

---

## References

- Interactive Curve Visualization: [Desmos Simulation](https://www.desmos.com/calculator/ieknhex0lj)
- NumPy Mathematical Functions Documentation: [https://numpy.org/doc/stable/reference/routines.math.html](https://numpy.org/doc/stable/reference/routines.math.html)
- pandas `read_csv` Documentation: [https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html](https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html)
- SciPy `differential_evolution` Documentation: [https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.differential_evolution.html](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.differential_evolution.html)
- SciPy `least_squares` Documentation: [https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.least_squares.html](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.least_squares.html)
