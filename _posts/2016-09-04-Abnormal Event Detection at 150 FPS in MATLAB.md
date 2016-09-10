---
layout:     post
title:      "Abnormal Event Detection at 150 FPS in MATLAB"
subtitle:   "论文笔记"
date:       2016-09-04
author:     "icbcbicc"
header-img: "img/post-bg-04.jpg"
tags: []

---

# Abnormal Event Detection at 150 FPS in MATLAB

## Introduction

#### Difficulties in detecting abnormal events based on surveillance videos

- Hard to list all possible negative samples  

#### Traditional method

- Normal patterns are learned from training

- And then the patterns are used to detect events deviated from this representation

#### Other methods

- Trajectories extracted from object-of-interest are used as normal patterns

- Learn normal low-level video feature distributions(exponential, multivariate Gaussian mixture, clustering)

- Graph model normal event representations which use co-occurrence patterns

- sparsity-based model(not fast enough for realtime processing due to the inherently intensive computation to build sparse representation)

## Sparsity Based Abnormality Detection

- ***Sparsity:  A general constraint to model normal event patterns as a linear combination of a set of basis atoms***

- ***Computationally expensive***

	$$ min\|x-D\beta\|^2_2 \qquad s.t. \qquad \|\beta\|_0 \le s $$

	 $\beta$ : parse coefficients  

	 $\|x-D\beta\|^2_2 $ :data fitting term  

	 $\|\beta\|_0$ : sparsity regulization term  

	 $s$ : parameter to control sparsity  

	- abnormal pattern : large error result from $\|x-D\beta\|^2_2$


- ***Efficiency problem***

	Adopting $min\|x-D\beta\|^2_2$ is time-consuming :  

    - **ﬁnd the suitable basis vectors** (with scale $s$) from the dictionary (with scale $q$) to represent testing data $x$)

	- Search space is large **( $(^q_s)$ different combinations )**

- ***Our contribution***
	- Instead of coding sparsity by ﬁnding an $s$ basis combination from $D$ in $min\|x-D\beta\|^2_2$, we code it directly **as a set of possible combinations of basis vectors**

    -  We only need to ﬁnd the most suitable combination by evaluating **the small-scale least square error**

    ![Our testing architecture](/img/3.JPG)

    - Freely selecting $s$ basis vectors from a total of $q$ vectors, the reconstructed structure could deviate from input due to the **large freedom**. However, in our method, **each combination ﬁnds its corresponding input data**

    -  It reaches 140∼150 FPS using a desktop with 3.4GHz CPU and 8G memory in MATLAB 2012.

## Methods

#### Learning Combinations on Training Data

- ***Preprocess***
	-  Resize each frame into different scales as "[Sparse Reconstruction Cost for Abnormal Event Detection](http://101.110.118.63/www3.ntu.edu.sg/home/jsyuan/index_files/papers/Cong-Yuan-CVPR11.pdf)"

    - Uniformly partition each layer to a set of non-overlapping patches. All patches have the same size

    - Corresponding regions in 5 continuous frames are stacked together to form a spatial-temporal cube.

    - ![Resize](/img/4.JPG)

    This pyramid involves **local information** in ﬁne-scale layers and more **global structures** in small-resolution ones.

    - With the spatial-temporal cubes, we compute **3D gradient features** on each of them following "[Anomaly Detection in Extremely Crowded Scenes Using Spatio-Temporal Motion Pattern Models](https://www.cs.drexel.edu/~kon/publication/LKratz_CVPR09.pdf)"

	- Features are processed separately according to their spatial coordinates. **Only features at the same spatial location in the video frames are used together** for training and testing.

- ***Learning Combinations on Training Data***
	- **Reconstruction Error**
 $$t=min_{s,\gamma,\beta}\sum_{j=1}^n\sum_{i=1}^K\gamma_j^i\|x_j-S_i\beta_j^i\|_2^2 \quad s.t. \sum_{i=1}^K\gamma_j^i=1,\gamma_j^i=\{0,1\}$$

	- Each $\gamma_j^i$ indicates whether or not the $i^{th}%$ combination $S_i$ is chosen for data $x_j$ and **only one combination is selected**

	- **$K$ must be small enough** based on redundant surveillance video information because **a very large $K$ could possibly make the reconstruction error $t$ always close to zero**, even for abnormal events

- ***Optimization for Training***

	- **Problem : ** **Reducing $K$ could increase reconstruction errors $t$**. And it is not optimal to ﬁx $K$ as well,as content may vary among videos

	- **Solution : **  A maximum representation strategy. It **automatically ﬁnds $K$ while not wildly increasing the reconstruction error $t$**. In fact, error $t$ for each training feature is upper bounded in our method

	- **Updated function**
	 $$t_j=\sum_{i=1}^K\gamma_j^i \{ \|x_j-S_i\beta_j^i \|_2^2 -\lambda \} \le 0,\quad s.t. \sum_{i=1}^K\gamma_j^i=1,\gamma_j^i=\{0,1\}$$

	- $\lambda ： $  **Reconstruction error upper bound** uniformly for all elements in $S$. If $t_j$ is smaller than $\lambda$, the coding result is with good quality

- ***Algorithm***

	- **In each pass, we update only one combination**, making it represent as many training data as possible

	- This process can **quickly ﬁnd the dominating combinations encoding important and most common features**

	- Remaining training cube features that cannot be well represented by this combination are **sent to the next round to gather residual maximum commonness**

	- This process **ends until all training data are computed and bounded**

	- The size of combination $K$ reﬂects how informative the training data are

	- In each pass, we **solve the equation above by interatively update $\{S_s^i,\beta\}$ and $\lambda$**
		1. **Update $\{S_s^i,\beta\}$** :

		$$L(\beta,S_i)=\sum_{j \in \Omega_c}\gamma_j^i \{ \|x_j-S_i\beta_j^i \|_2^2 \}$$  

		$$\beta_j^i=(S_i^TS_i)^{-1}S_i^Tx_j$$  

		(Optimize $\beta$ while ﬁxing $S_i$ for all $\gamma_j^i \neq 0$)

		$$ S_i=\prod[S_i-\delta_t\bigtriangledown_{S_i}L(\beta,S_i)]$$

		( Using **block-coordinate descent**( [Online learning for matrix factorization and sparse coding](http://www.jmlr.org/papers/volume11/mairal10a/mairal10a.pdf) and set $\delta = 1E-4$)

		2. ** Update $\gamma$ ** :

		$$\gamma_j^i = \left\{
		\begin{aligned}
		1 \quad & if \quad \|x_j-S_i\beta_j^i \|_2^2 < \lambda ; \\
		0 \quad & otherwise ;\\
		\end{aligned}
		\right .$$

	![Algorithm 1](/img/5.JPG)

    **The algorithm is controlled by $\lambda$, Reducing it could lead to a larger $K$**

- ***Testing***

	- With the learned sparse combinations $S = \{S_1, ..., S_K\}$, in the testing phase with new data $x$, we checki f there exists a combination in $S$ ﬁtting the $\lambda$. It can be quickly achieved by checking the least square error for each $S_i$   
	
	$$min_{\beta^i}\|x_j-S_i\beta_j^i \|_2^2 \quad \forall i= 1, ..., K$$

	- **optimal solution :**  

	$$\hat{\beta^i}=(S_i^TS_i)^{-1}S_i^Tx$$

	- **Reconstruction error in $S_i$ :**

	$$\|x_j-S_i\beta_j^i \|_2^2 = \|((S_i^TS_i)^{-1}S_i^T-I_p)x\|_2^2=\|R_ix\|_2^2$$

	![Algorithm 2](/img/6.JPG)

	- It is noted that the ﬁrst a few dominating combinations represent the largest number of normal event features.

	- Easy to parallel to achieve $O(1)$ complexity

	- Average combination checking ratio $= \frac{The\  number\ of\ combinations\ checked}{The\ total\ number\ K}$

- ***Relation to Subspace Clustering***

## Experiments