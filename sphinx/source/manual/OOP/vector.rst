.. _manual-OOP-vector:

#########
 Vector
#########

This routine defines a vector field, such as velocity. This is simply an array holding the two or three component fields. This is more convient than scattering small arrays around the code and gives a place to hold vector specific routines like divergence and gradient.

* :ref:`initv`
* :ref:`divergence`
* :ref:`gradient`
* :ref:`gradTensor`
* :ref:`vorticity`
* :ref:`lambda2`

.. _initv:

**********
Initialize
**********

.. function:: init(a,n,V,b,exit,l)

    initializes a vector field, that contains 3 different scalar field (one for each component of the vector)


.. _divergence:

***********
 Divergence
***********

.. function:: divergence(v,f)

    Takes the divergence of a specified vector field

    :param v: Vector to take divergence from
    :type v: Class[vector]
    :param f: divergence
    :type f: Class[field]

Takes the divergence of a the field ``f``. Divergence of a vector field is  the sum of the flux on each cell face by Gauss theorem. On a uniform grid this is easily evaluated. On a rectilinear grid, we must scale each of the flux by the face area

.. math::
    :nowrap:

    \begin{equation}
        \int\nabla\cdot\vec{u} \, dV = \oint \vec{u}\cdot d\vec{n} \simeq \sum_{d=1}^{Ndims}(u_{i+1/2} - u_{i-1/2})(A_i) = Vol_j\sum_{d=1}^{Ndims} \frac{u_{i+1/2} - u_{i-1/2}}{dx_i}
    \end{equation}

Where we essentially have the flux at each face and the volume of the j-th cell divided by the i-th dimension of the cell, gives us the orentiend area in the i-th direction.


.. note::
    For a given set of index ``(i,j,k)`` the velocity values are located to the left/bellow/front of the corresponding pressure point in physical space. If we say cell centers are located at integers values of ``(i,j,k)``, the divergence uses velocity points with ``(i,j,k)`` and ``(i+1,j,k)`` and the gradient uses scalar at index ``(i,j,k)`` and ``(i-1,j,k)``.

.. _gradient:

*********
 Gradient
*********

.. function:: gradient(v,p) 

    Takes the gradient of a specified scalar field

    :param v: Vector of gradient of p
    :type v: Class[vector]
    :param p: field to take gradient of
    :type p: Class[field]

The i-th component of the gradient of a scalar field is computed as

.. math::
    :nowrap:

    \begin{equation}
        [\nabla p]_i = \frac{\partial p}{\partial x_i} \simeq \frac{p_{i,j,k} - p_{i-1,j,k}}{h} = \frac{2(p_{i,j,k} - p_{i-1,j,k})}{dx_{i,j,k}+dx_{i-1,j,k}}
    \end{equation}

where h is the distance between two scalar cell centers. This is simply the average of the neigbouring vector cells ``dx(i,j,k)`` and ``dx(i-1,j,k)`` (this is already computed as ``dxi2`` in the code).


.. _gradTensor:

****************
 Tensor Gradient
****************


.. function:: gradTensor(v,S)

    Takes the gradient of a vector field and return a secon-order tensor

    :param v: Vector to take gradient of
    :type v: Class[vector]
    :param S: gradient tensor of vector v
    :type S: Class[field(3)]

The velocity gradient tensor is returned in the array ``S`` that is passed to the subroutine.

.. math::
    :nowrap:

        \begin{equation}
            \nabla\vec{v} = \frac{\partial v_i}{\partial x_j}
        \end{equation}

Each component is a cell-centered vector. In-line terms are easily evaluated

.. code:: fortran

    S(d)%e(d)%point() = v%e(d)%point(d)-v%e(d)%point()

but cross terms require interpolation once the gradient has been calculated.

.. code:: fortran

    o = (/0,0,0/)
    o(d2) = -1; v00 => v%e(d1)%point(arg=o)
    o(d1) =  1; v01 => v%e(d1)%point(arg=o)
    o(d2) =  1; v11 => v%e(d1)%point(arg=o)
    o(d1) =  0; v10 => v%e(d1)%point(arg=o)
    S(d1)%e(d2)%point()= 0.25*(v10-v00+v11-v01)

.. note::
    ``S(i)%e(j)%p`` stores the value of : :math:`\partial u_i/\partial x_j`


.. _vorticity:

***********
 Vorticity
***********

.. function:: vorticity(uij, omega)

    Computes the vorticity field, given a velocity gradient tensor

    :param uij: velocity gradient tensor
    :type uij: Class[vector(3)]
    :param omega: vorticity field
    :type omega: Class[vector]

The vorticity is return in the array ``omega`` that is passed to the subroutine. Voriticity is a cell-centered vector.

.. math::
    :nowrap:

    \begin{equation}
        \vec{\omega} = \nabla\times\vec{u} \rightarrow \omega_i = \varepsilon_{ijk}\frac{\partial}{\partial x_j}u_k
    \end{equation}


.. _lambda2:

******************
 :math:`\lambda_2`
******************

.. function:: lambda2(uij, lam2)

    Computes the lambda-2 criterion of a given velocity gradient tensor ``uij``

    :param uij: velocity gradient tensor
    :type uij: Class[vector(3)]
    :param lam2: lambda-2 criterion
    :type lam2: Class[field]

The :math:`\lambda_2` criterion is the second larges (in magnitude) eigenvalue of

.. math::
    :nowrap:

    \begin{equation}
        S_{ik}S_{kj} + \Omega_{ik}\Omega_{kj}
    \end{equation}

