.. _manual-OOP-MGsolver:

#########
 MGsolver
#########

Holds the data and routines to approximately invert the linear system arising from a Poisson equation using a Multi-Grid scheme. The key routines are init and update. Like all MG methods, the residual is down-sampled to successively coarser grids and the correction up-sampled to keep convergence from stalling. Somewhat uniquely, this solver uses a single Jacobi step before down-sampling, and a Conjugate Gradient method instead of a smoother after up-sampling. Simple V-cycles are used until converged. 

The default maximum number of V-Cycle is 32, with an average tolerence between scuccesive iterations on :math:`1e-6`. A maximum if 32 PCG iterations are perfomed at each level of the multi-grid.

* :ref:`initmg`

.. _initmg:

***********
 Initialize
***********

.. function:: init(a,beta)

    Iitializes a multi-grid solver

    :param a: MGsolver
    :type a: Class[MGsolver]
    :param beta: diagonal coefficient
    :type beta: Class[vector]

Builds the multi-gird solver. Because the Poisson matrix is tri-diagonnal andsymmetric, we only need to store the lower and diagonal coefficients.
