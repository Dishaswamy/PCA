# Background Subtraction using PCA and Robust PCA

Background subtraction aims to separate the relatively persistent **background** of a video from moving **foreground objects**.

This project studies background subtraction from a low-rank matrix decomposition perspective and is divided into two stages:

- **Part A — PCA / Truncated SVD:** Model a static background using a low-rank approximation of the video matrix.
- **Part B — Robust PCA:** Decompose the video into low-rank background and sparse foreground components.

---

# Part A — Background Subtraction using PCA

## 1. Mathematical Approach

Consider a video consisting of $n$ grayscale frames, where each frame contains $d$ pixels.

Each frame is flattened into a vector

$$
x_i \in \mathbb{R}^{d}
$$

and the frame vectors are stacked as columns of a matrix

$$
X =
\begin{bmatrix}
x_1 & x_2 & \cdots & x_n
\end{bmatrix}
$$

where

$$
X \in \mathbb{R}^{d \times n}.
$$

Thus, each column of $X$ represents one video frame.

For a video captured using a stationary camera, a large portion of the scene remains approximately unchanged across frames. Consequently, the frame vectors are highly correlated.

This means that although each frame contains $d$ pixel values, most of the persistent structure of the video can often be represented using a much smaller number of components.

Therefore, the background is expected to have an approximately **low-rank structure**.

Instead of representing the video using the complete matrix $X$, we seek a rank-$k$ approximation

$$
X \approx X_k
$$

where $k$ is much smaller than the dimensions of the original matrix.

The low-rank approximation is expected to capture the dominant structures shared across many frames, which primarily correspond to the stationary background.

---

## 2. Singular Value Decomposition

The video matrix is decomposed using Singular Value Decomposition (SVD):

$$
X = U \Sigma V^T
$$

where:

- $U$ contains the left singular vectors,
- $\Sigma$ contains the singular values,
- $V$ contains the right singular vectors.

The singular values are ordered as

$$
\sigma_1 \geq \sigma_2 \geq \sigma_3 \geq \cdots \geq 0
$$

Large singular values correspond to components that explain a larger amount of the structure present in the video.

Keeping only the first $k$ singular values and their corresponding singular vectors gives the truncated SVD:

$$
X_k = U_k \Sigma_k V_k^T
$$

where:

- $U_k$ contains the first $k$ left singular vectors,
- $\Sigma_k$ contains the first $k$ singular values,
- $V_k$ contains the corresponding right singular vectors.

By the **Eckart–Young theorem**, truncated SVD gives the best rank-$k$ approximation of $X$ under the Frobenius norm.

In other words, among all matrices having rank at most $k$, $X_k$ minimizes

$$
\|X-X_k\|_F^2
$$

The squared Frobenius reconstruction error can be calculated directly from the singular values that were discarded:

$$
\|X-X_k\|_F^2
=
\sum_{i=k+1}^{r}\sigma_i^2
$$

where $r$ is the number of singular values.

Therefore, the reconstruction error can be calculated without explicitly reconstructing $X_k$ for every value of $k$.

---

## 3. Interpretation for Background Subtraction

The dominant singular components describe patterns that occur repeatedly across the video frames.

For a stationary-camera sequence, the background appears in approximately the same location across many frames. It therefore contributes strongly to the dominant singular components.

The video matrix can be approximately viewed as

$$
X = L + E
$$

where:

- $L$ represents the low-rank background,
- $E$ represents deviations from that background.

Using truncated SVD, the background is approximated as

$$
L \approx X_k
$$

with

$$
X_k = U_k \Sigma_k V_k^T
$$

The residual between the original video and its low-rank approximation is

$$
E = X-X_k
$$

Pixels belonging to moving objects tend to differ from their low-rank reconstruction and therefore produce larger values in the residual.

This residual can consequently be used to identify moving foreground objects.

---

# Dataset

The experiments use the **Highway** sequence from the **CDnet 2014 (Change Detection 2014)** dataset, obtained through its Kaggle distribution.

The Highway sequence belongs to the **Baseline** category and contains traffic recorded by a stationary camera overlooking a highway.

It provides a suitable setting for demonstrating PCA-based background subtraction because:

- the camera remains stationary,
- the road and surrounding environment remain relatively consistent across frames,
- the major temporal changes are caused by moving vehicles.

As a result, the persistent background has an approximately low-rank structure, while moving vehicles appear as deviations from this structure.

The dataset also provides ground-truth foreground masks, which can be used to evaluate the predicted segmentation masks.

---

# Implementation

## 1. Constructing the Video Matrix

The input frames are loaded and converted to grayscale.

Each frame is flattened into a one-dimensional vector

$$
x_i \in \mathbb{R}^{d}
$$

The flattened frames are then stacked as columns to construct the video matrix

$$
X \in \mathbb{R}^{d \times n}
$$

where $d$ is the number of pixels in each frame and $n$ is the number of video frames.

Thus,

$$
X =
\begin{bmatrix}
x_1 & x_2 & \cdots & x_n
\end{bmatrix}
$$

---

## 2. Computing the SVD

SVD is applied to the video matrix:

$$
X = U \Sigma V^T
$$

The resulting singular values describe how strongly each component contributes to the representation of the video.

The singular values are arranged in decreasing order:

$$
\sigma_1 \geq \sigma_2 \geq \cdots
$$

For a video containing a mostly static background, a relatively small number of dominant singular components is expected to capture much of the persistent structure of the scene.

---

## 3. Selecting the Rank $k$

The choice of $k$ determines how many singular components are retained in the background model.

For each candidate value of $k$, the squared Frobenius reconstruction error is calculated:

$$
E(k) = \|X-X_k\|_F^2
$$

Using the singular values, this can be calculated as

$$
E(k)
=
\sum_{i=k+1}^{r}\sigma_i^2
$$

The reconstruction error is then plotted against $k$.

For small values of $k$, increasing the rank generally produces a large decrease in reconstruction error. As more components are retained, the improvement eventually becomes much smaller.

The **knee or elbow point** of the reconstruction-error curve provides a practical choice for $k$.

This gives a trade-off between retaining enough components to accurately model the background and avoiding unnecessary components that may begin to capture foreground variation.

---

## 4. Background Reconstruction

After selecting $k$, the rank-$k$ approximation of the video matrix is reconstructed as

$$
X_k = U_k \Sigma_k V_k^T
$$

Each column of $X_k$ corresponds to the reconstructed version of one video frame.

The reconstructed column is reshaped back into the original image dimensions to obtain an estimate of the background frame.

Because only the dominant singular components are retained, structures that remain persistent across frames, such as the road and surrounding environment, are captured by the reconstruction.

Transient structures such as moving vehicles are not represented as effectively by this low-rank model.

---

## 5. Foreground Extraction

The absolute difference between the original video matrix and its low-rank reconstruction is calculated:

$$
D = |X-X_k|
$$

A threshold $\tau$ is applied to this difference.

A pixel is classified as foreground when

$$
D_{ij} > \tau
$$

and as background when

$$
D_{ij} \leq \tau
$$

This produces a binary foreground mask in which pixels with sufficiently large reconstruction errors are identified as moving objects.

Different values of $\tau$ can be tested to study the effect of the threshold on foreground segmentation.

The predicted foreground masks can then be compared against the ground-truth masks using metrics such as:

- **Precision**
- **Recall**
- **F1 Score**

---

# Limitations of Classical PCA

Classical PCA works well when the video contains a mostly static background that can be represented using a low-dimensional subspace.

However, the method becomes less reliable when the video contains large foreground changes, persistent foreground objects, outliers, or significant background motion.

## Squared Error and the Gaussian Noise Model

PCA finds a low-rank representation by minimizing squared reconstruction error.

For a rank-$k$ approximation, the objective is to make

$$
\|X-X_k\|_F^2
$$

as small as possible.

More generally, if $L$ represents the low-rank approximation, the squared Frobenius error is

$$
\|X-L\|_F^2
=
\sum_{i,j}(X_{ij}-L_{ij})^2
$$

Define the residual matrix as

$$
E = X-L
$$

Then the objective becomes

$$
\|E\|_F^2
=
\sum_{i,j}E_{ij}^2
$$

This least-squares formulation has an important statistical interpretation.

It corresponds to modeling the residual entries as independent Gaussian noise:

$$
E_{ij} \sim \mathcal{N}(0,\sigma^2)
$$

This does **not** mean that PCA requires the original data matrix $X$ to follow a Gaussian distribution.

Rather, the squared-error formulation corresponds to a Gaussian model for the reconstruction errors or residual noise.

This is reasonable when the deviations from the low-rank representation are small random perturbations.

Foreground objects in a video behave differently.

A moving vehicle can cause large intensity differences over a relatively small subset of pixels. Such changes are better interpreted as **large but sparse deviations** from the background rather than small Gaussian perturbations.

Because PCA uses squared reconstruction error, a residual contributes to the objective according to

$$
E_{ij}^2
$$

Therefore, large residuals receive a disproportionately large penalty.

Strong or persistent foreground objects can consequently influence the principal components themselves. Instead of being completely excluded from the background model, parts of foreground objects may be absorbed into the low-rank reconstruction.

This is one reason why classical PCA is sensitive to outliers and sparse foreground changes.

---

## Why PCA Struggles with Dynamic Backgrounds

The static-background assumption becomes more restrictive when the background itself changes over time.

Examples of dynamic background motion include:

- moving tree branches,
- flowing water,
- waves,
- fountains,
- changing illumination,
- repetitive environmental motion.

For a mostly static scene, the video matrix can be approximated as

$$
X = L + E
$$

where $L$ represents the persistent low-rank background and $E$ contains deviations from it.

When the background itself moves, however, the background pixels also change across frames.

These changes may require additional components to represent and may not be captured effectively by a small rank-$k$ approximation.

Consequently, the residual

$$
E = X-X_k
$$

can contain two different types of variation:

1. **Foreground motion**, such as vehicles or people.
2. **Background motion**, such as moving leaves, water, or illumination changes.

The foreground detection method relies on thresholding the magnitude of this residual.

Therefore, when dynamic background regions produce large reconstruction errors, they may be incorrectly classified as foreground.

This increases false positives and reduces the quality of the foreground segmentation.

There is also a more fundamental limitation: classical PCA does not explicitly distinguish between the low-rank background and large sparse foreground deviations. Both affect the same least-squares reconstruction.

Ideally, we would like to model these components separately:

$$
X = L + S
$$

where:

- $L$ represents the **low-rank background**,
- $S$ represents **sparse foreground deviations**.

This motivates **Robust PCA**, which explicitly separates the low-rank and sparse components of the video matrix.

Robust PCA forms **Part B** of this project.
