# AI Evasion
Hands-on adversarial machine learning notebooks from the HTB AI Red Teamer Path, with implementations and notes for FGSM, PGD, DeepFool, ElasticNet/FISTA, and related evasion techniques.


# AI Evasion – Adversarial Machine Learning

Hands-on adversarial machine learning work from the **Hack The Box AI Red Teamer Path**, focused on understanding and implementing evasion attacks against machine learning models.

This repository contains Jupyter notebooks, experiments, and study notes for attacks including FGSM, PGD, DeepFool, ElasticNet, FISTA-based optimization, and related adversarial techniques.

## Goals

The goal of this repository is not just to reproduce attack code, but to understand:

* how adversarial attacks formulate optimization problems
* how perturbation constraints such as L0, L1, L2, and L∞ differ
* how gradients are used to manipulate model predictions
* how adversarial loss functions drive misclassification
* how optimization methods such as projected gradient descent and FISTA work
* how sparsity, smoothness, confidence, and perturbation magnitude trade off against one another
* how the mathematical formulations map directly to PyTorch implementations

## Current Topics

### FGSM

Fast Gradient Sign Method

Key concepts:

* gradient-based adversarial perturbations
* L∞ constraints
* epsilon-bounded attacks
* sign of the loss gradient

### PGD

Projected Gradient Descent

Key concepts:

* iterative gradient attacks
* projection back into a constrained perturbation region
* attack strength versus perturbation budget

### DeepFool

Key concepts:

* local linearization of neural-network decision boundaries
* geometric projection
* minimum-distance adversarial perturbations
* L2-based attacks

### ElasticNet / FISTA

Key concepts:

* mixed L1 and squared-L2 penalties
* sparse adversarial perturbations
* Carlini & Wagner margin-based adversarial loss
* proximal operators
* soft thresholding
* proximal gradient descent
* FISTA and Nesterov-style momentum
* per-example optimization
* binary search over attack trade-off constants

## ElasticNet Objective

ElasticNet seeks an adversarial image (x') that fools the classifier while minimizing perturbation size.

The perturbation is:

[
\delta = x' - x
]

where:

* (x) is the original image
* (x') is the adversarial image
* (\delta) is the perturbation

The elastic-net distance combines squared L2 and L1 penalties:

[
|\delta|_2^2 + \beta|\delta|_1
]

The squared L2 term discourages large individual changes, while the L1 term encourages sparse perturbations in which many pixels remain unchanged.

The optimization also includes an adversarial loss that pushes the model toward misclassification.

FISTA handles the objective by separating it into:

* a **smooth component**, optimized using gradients
* a **non-smooth L1 component**, handled using a proximal operator and soft thresholding

## Learning Approach

These notebooks are being developed as both implementations and study material.

For each attack, the workflow is generally:

1. understand the attack objective
2. break down the mathematical notation
3. translate the equations into plain English
4. implement the corresponding PyTorch operations
5. run the attack against a model
6. inspect perturbation magnitude and attack success
7. compare the technique with previously studied attacks

Particular emphasis is placed on understanding how mathematical expressions translate into code rather than treating equations as black boxes.

## Repository Structure

The repository will evolve as the HTB AI Red Teamer Path progresses.

Example structure:

```text
AI_evasion/
├── FGSM.ipynb
├── PGD.ipynb
├── DeepFool.ipynb
├── ElasticNet.ipynb
├── output/
│   └── model checkpoints and experiment outputs
└── README.md
```

Notebook names and supporting files may change as additional techniques are implemented.

## Environment

The work is primarily performed with:

* Python
* Jupyter Notebook
* PyTorch
* NumPy
* Hack The Box AI course utilities

A Conda environment is used for the project.

Example:

```powershell
conda activate ai
jupyter notebook
```

## Status

This repository is a work in progress and follows the progression of the HTB AI Red Teamer curriculum.

Current focus:

**ElasticNet adversarial attacks and FISTA optimization**

## Disclaimer

This repository is for educational, defensive security, and authorized adversarial machine learning research.

All testing should be performed against systems and models that you own or have explicit permission to evaluate.
