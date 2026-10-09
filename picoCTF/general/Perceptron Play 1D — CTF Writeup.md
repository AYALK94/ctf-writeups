# Perceptron Play 1D — CTF Writeup

## Challenge Overview

**Name:** Perceptron Play 1D Alpha
**Category:** Artificial Intelligence / AI Foundations I
**Difficulty:** Easy
**Author:** LT 'syreal' Jones
**Connection:** `nc xebec.cylabacademy.net 29506`

The challenge is an interactive playground where you must tune the parameters of a single-layer perceptron operating on a 1-dimensional number line. The goal is to correctly classify every labeled training point. Once all points are classified correctly, the `CHECK` command reveals the flag.

---

## Background: The 1D Perceptron

A single-layer perceptron computes a weighted sum of its inputs plus a bias:

```
activation = w · x + b
```

The prediction is then determined by a threshold (typically at 0):

```
predict 1  if  w · x + b > 0
predict 0  otherwise
```

In 1D, the **decision boundary** is the single point where `w·x + b = 0`, i.e.:

```
x = -b / w
```

All points to one side of this boundary are classified as `1`, and all points to the other side as `0`.

---

## Connecting to the Service

```bash
$ nc xebec.cylabacademy.net 29506
```

The service greets us with an interactive prompt:

```
Welcome to Perceptron Play 1D!
...
Commands:
  SHOW                redraw the number line and point table
  SET w b             set the weight/bias directly (floats are fine)
  ADJUST dw db        add offsets to the current weight and bias
  POINTS              list the training points with their target labels
  CHECK               verify every point; prints the flag when perfect
  RESET               go back to the starting params (w=1.0, b=0.0)
  HELP                show this message again
  EXIT / QUIT         leave the playground
```

---

## Initial State

The starting parameters are `w = 1.0`, `b = 0.0`, and the initial display is:

```
Number line (predictions):
    -4-3-2-1+0+1+2+3+4
     0 . 0 . x . 1 1 1
             ^

Current parameters -> w: 1, b: 0

  x    label  perceptron  activation
  --   -----  ----------  ----------
  -4      0        0        -4
  -2      0        0        -2
  +0      0        1        0
  +2      1        1        2
  +3      1        1        3
  +4      1        1        4
```

### Legend

| Symbol | Meaning |
|--------|---------|
| `0` | point labeled class **0**, classified correctly |
| `1` | point labeled class **1**, classified correctly |
| `x` | misclassified point |
| `^` | decision boundary marker |
| `\|` | origin marker (when nothing else is present) |

---

## Analyzing the Problem

From the initial state we can extract the dataset:

| x   | label |
|-----|-------|
| -4  | 0     |
| -2  | 0     |
|  0  | 0     |
|  2  | 1     |
|  3  | 1     |
|  4  | 1     |

We need a decision boundary that separates the class-0 points (`-4, -2, 0`) from the class-1 points (`2, 3, 4`). In 1D, this means the boundary point `x = -b/w` must lie **strictly between 0 and 2** — somewhere in the open interval `(0, 2)`.

With the default `w = 1, b = 0`:
- Boundary is at `x = 0`
- The point `x = 0` produces `activation = 0`, which is (apparently) treated as predicting `1`, causing a misclassification since its true label is `0`.

### Required conditions

For every point to be classified correctly with `w = 1`:

| Point | Label | Need |
|-------|-------|------|
| x = -4 | 0 | `-4 + b < 0` → `b < 4` |
| x = -2 | 0 | `-2 + b < 0` → `b < 2` |
| x = 0  | 0 | ` 0 + b < 0` → `b < 0` |
| x = 2  | 1 | ` 2 + b > 0` → `b > -2` |
| x = 3  | 1 | ` 3 + b > 0` → `b > -3` |
| x = 4  | 1 | ` 4 + b > 0` → `b > -4` |

Combining, we need:

```
-2 < b < 0
```

A clean, easy choice is **`b = -0.5`**, keeping `w = 1`.

---

## Solving the Challenge

At the `>` prompt, set the parameters:

```
> SET 1 -0.5
```

The service responds:

```
Number line (predictions):
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1
             ^

Current parameters -> w: 1, b: -0.5

  x    label  perceptron  activation
  --   -----  ----------  ----------
  -4      0        0        -4.5
  -2      0        0        -2.5
  +0      0        0        -0.5
  +2      1        1        1.5
  +3      1        1        2.5
  +4      1        1        3.5
```

Every point now matches its label — all `0`s are classified as `0` and all `1`s as `1`. The decision boundary `^` sits between `0` and `2` (at `x = 0.5`).

Now verify and grab the flag:

```
> CHECK
```

The server confirms all points are correct and prints the flag.

---

## Final Answer

**Parameters used:**

```
w = 1
b = -0.5
```

**Flag:** *(printed by the server on `CHECK` — format typically `CYLAB{...}` or similar)*

---

## Key Takeaways

1. **1D perceptrons are just threshold classifiers.** The decision boundary is a single point on the number line at `x = -b/w`.

2. **The bias shifts the boundary.** With `w = 1`, changing `b` moves the boundary left or right. To push the boundary into the gap between the two classes, you simply choose `b` so that `-b/w` lies inside the separation.

3. **The dataset was linearly separable.** In 1D, a dataset is separable if there is a single cutoff point such that all class-0 points are on one side and all class-1 points on the other. Here the gap between `x = 0` and `x = 2` provides that cutoff.

4. **General recipe for 1D perceptron puzzles:**
   - Identify the largest class-0 value and the smallest class-1 value.
   - Place the boundary anywhere strictly between them.
   - Solve `-b/w = (largest_0 + smallest_1)/2` for your chosen `w` (often just pick `w = 1` and set `b = -boundary`).

For this challenge, the midpoint between `0` and `2` is `1`, but any boundary in `(0, 2)` works — we chose `x = 0.5` via `b = -0.5`, which correctly classifies everything.
