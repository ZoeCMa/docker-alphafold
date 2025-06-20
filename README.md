Docker images for Alphafold 2
=============================

The directories contain two Docker build recipies for Alphafold 2 --

- docker-alphafold-rocm: Alphafold with AMD ROCm support for the underlying Tensorflow and Jax libraries.
- docker-alphafold-no-rocm: Alphafold without ROCm support.

Alphafold is checked out at the Git commit 020cd6d6cb16540114a084f9dbb8f21f811f9d21.

The images contain the following dependencies --

- HH-Suite "master" branch at Git commit bf3f42747ca4ddbd72e1c142aa3b648793c59045
- OpenMM v8.0.0
- pdbfixer v1.9.0
- tensorflow v2.18
- jax v0.6.0

The Python version is 3.12 for both images.

ROCm notes
----------

The ROCm-enabled image is based on version 6.4.1 Binaries are compiled for the
GPU architecture gfx1100. The environment variable
`HSA_OVERRIDE_GFX_VERSION=11.0.0` may be defined to force the architecture in
use at runtime.
