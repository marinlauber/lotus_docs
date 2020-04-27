.. _manual-OOP-fluid:

#######
 Fluid
#######

The fluid module is the primary building-block of Lotus. It holds the velocity and pressure fields, as well as the Poisson solver data. It also contains subroutines that initalize, update and write different attributes of the fluid attributes. 

The following shows the different attributes and procedures that are contained in the fluid module.

.. code:: fortran

   module fluidMod
      use fieldMod, only: field
      use vectorMod, only: vfield
      use MGsolverMod, only: MGsolver
      use bodyMod, only: body
      !
      ! -- Basic fluid data type
      type :: fluid
         type(vfield)   :: velocity         ! velocity field
         type(field)    :: pressure         ! pressure field
         type(MGsolver) :: psolver          ! Poisson solver
         type(field)    :: sigma            ! pressure source term
         type(vfield)   :: u0,force         ! old velocity and force
         real           :: time,dt          ! time and time step
         real           :: nu=0             ! kinematic viscosity
         real           :: g(3)=0           ! gravitational vector
         logical        :: body=.false.     ! body in flow?
         type(vfield)   :: mu0,ub           ! body kernel coefficients and velocity
         type(vfield),allocatable :: mu1(:) ! moment correction coeffs
         contains
         procedure :: init, update, write, reset_u0
      end type fluid
   end module fluidmod

* :ref:`initialise`
* :ref:`update`
* :ref:`write`
* :ref:`reset`

.. _initialise:

**************
 initalise
**************

This initalises the fluid class. 

.. function:: init(a,n,geom,V,nu,NNin,exit)
   
    Initalises the fluid class

    :param a: fluid class to initialize
    :type a: fluid
    :param n: number of cells in each direction
    :type n: int
    :param geom: geometry class to use for update
    :type geom: Optional[body]
    :param V: initial conditions, default is zero background velocity
    :type V: Optional[real]
    :param nu: fluid viscosity
    :type nu: Optional[real]
    :param NNin: starting contition
    :type NNin: Optional[int]
    :param exit: convection exit condition
    :type exit: Optional[logical] 
    :rtype: N-A

The size of the field passed to the subroutine ``n(3)`` is the local field of each of the MPI block. This means that if ``m(3)`` is the global domain dimensions and ``b(3)`` is the MPI domain decomposition, the flow is initialized as follows

.. code:: fortran
   flow%init(m/b,geom,V=(/1.0,0.0,0.0/),nu)

for a given ``geom`` and ``nu``.

When starting a simultion from another one, the ``NNin`` parameter can be set to take this specific fluid state, and not the most recent one. 

Relevant files within the repos directory:

* oop/fluid.f90

.. _update:

*******
 update
*******

This subtourine updates the fluid equations following 2-nd order BDIM.

.. function:: update(a,geom,V,b,f)
    
    Update the fluid equations

    :param a: fluid class to update
    :type a: fluid
    :param geom: geometry class to use for update
    :type geom: Optional[body]
    :param V: boundary conditions
    :type V: Optional[real]
    :param b: body force/acceleration
    :type b: Optional[real]
    :param f: source term
    :type f: Optional[vfield]
    :rtype: N-A

The fluid update uses a second-order (in time) predictor-corrector method (Heun's method) with a projection method to ensure that the resulting velocity field is divergence-free.

The update algortihm can be broken down into 4 steps: 

   1. update the geometry, and the multi-grid solver if needed. This is only called during intialization and if the body moves/changes

   .. code:: fortran

      if(present(geom).and..not.geom%upToDate) then
         call geom%measure(a%mu0,a%mu1,a%ub)
         call a%psolver%reinit(a%mu0)
      end if

   2.update boundary conditions

   .. code:: fortran

      forall(d=1:a%u0%ndims,present(V))
         a%velocity%e(d)%bound_val = V(d)
         a%g(d) = (V(d)-a%u0%e(d)%bound_val)/a%dt
      end forall

   3. add any body force term, this overwrites the acceleration

   .. code:: fortran

      forall(d=1:a%u0%ndims,present(b))
         a%g(d) = b(d)
      end forall

   4. update velocity and pressure using Heun's predictor-corrector

   .. code:: fortran

      do m=1,2 
         ...

   The first step in the flow update is to calculate the convective and viscous forces

   .. code :: fortran

         if(m==1) call a%force%convect_diffuse(a%u0,a%nu)
         if(m==2) call a%force%convect_diffuse(a%velocity,a%nu)

   such that ``a%force`` now contains both forces. Next the step is to add any forcing ``f`` or body force ``b``

   .. note::
      Here we are looping on each component of the vectors with ``e(d)``
   
   .. code :: fortran

         do d=1,a%u0%ndims
            u => a%force%e(d)%point()
            if(present(f)) u = u+f%e(d)%point()
            u = a%u0%e(d)%point()+a%dt*(u+a%g(d))

   Where we also have integrated this force with our initial condition and time-step. The next step is to aplly the DBIM equations. First we evaluate the term that is multplied by the first kernel moment (here application of BC allows MPI comunications to update halos)

   .. code:: fortran

            if(a%body) then
               u = u-a%ub%e(d)%point()
               call a%force%e(d)%applyBC

   .. note::
      ``a%force`` now stores 

      .. math::
         :nowrap:

         \begin{equation}
            \vec{u}_0 + \Delta t\left( - (\vec{u}_0\cdot\nabla)\vec{u}_0 + \nu\nabla^2\vec{u}_0\right)  - \vec{u}_b \simeq \vec{f} - \vec{b}
         \end{equation}

   Next, we can calculate the normal derivative of this field (stored in ``a%force``), mutliply it by ``a%mu1`` and store the result in ``a%sigma``

   .. code:: fortran

               call a%force%e(d)%ddn(a%mu1(d)%e,a%sigma)
            end if

   .. note::
      ``a%sigma`` now stores 

      .. math::
         :nowrap:

         \begin{equation}
            \mu_1\frac{\partial}{\partial n}(\vec{f} - \vec{b})
         \end{equation}

   Finally we apply both the BDIM on the flow, add the second-order correction term and we are done with the predictor

   .. code:: fortran

            u = a%mu0%e(d)%point()*u
            if(a%body) u = u+a%ub%e(d)%point()-a%sigma%point()
            u => a%velocity%e(d)%point()
            if(m==1) then                           
               u = a%force%e(d)%point()
            else
               u = 0.5*(u+a%force%e(d)%point()) 
            end if
         end do
         if(m==1) call a%velocity%applyBC(a%mu0,a%dt)
         if(m==2) call a%velocity%applyBC(a%mu0)

   Where the predicted velocity is now store in ``a%velocity`` and not in ``a%force`` anymore.

    .. note::
      ``a%velocity`` now stores 

      .. math::
         :nowrap:

         \begin{equation}
            (\vec{f} - \vec{b})\mu_0 + \vec{b} - \mu_1\frac{\partial}{\partial n}(\vec{f} - \vec{b}) = \mu_0\vec{f} + (1-\mu_0)\vec{b} - \mu_1\frac{\partial}{\partial n}(\vec{f} - \vec{b})
         \end{equation}

   Next we need to project out the divergent part of the velocity field. We know use ``a%sigma`` to store the divergent part of the velocity field (we must also correct the  divergence of the body if it changes volume)

   .. code:: fortran

         call a%velocity%divergence(a%sigma)
         if(present(geom)) call geom%divergence(a%sigma)

   Next we invert the Poisson equation, using the given sigma, and a guess pressure field

   .. code:: fortran

         a%pressure%p = a%pressure%p*a%dt/float(m)
         call a%psolver%update(a%pressure,a%sigma)
   
   This pressure field is the removed from the predicted velocity field to project-out it divergent part (this step over-write ``a%force`` with the pressure gradient)

   .. code:: fortran

         call a%force%gradient(a%pressure)
         do d=1,a%u0%ndims
            u => a%velocity%e(d)%point()
            u = u-a%mu0%e(d)%point()*a%force%e(d)%point()
         end do
         call a%velocity%applyBC(a%mu0)
         a%pressure%p = a%pressure%p/a%dt*float(m)
      end do

   .. note::
      ``a%velocity`` now stores the final velocity field 

      .. math::
         :nowrap:

         \begin{equation}
            \mu_0\vec{f} + (1-\mu_0)\vec{b} - \mu_1\frac{\partial}{\partial n}(\vec{f} - \vec{b}) - \Delta t\mu_0\nabla p
         \end{equation}

   Finally, we prepare the solver for the next update.

   .. code:: fortran

      a%time = a%time+a%dt
      a%dt = a%velocity%cfl_limit(a%nu)
      a%u0 = a%velocity


Relevant files within the repos directory:

* oop/fluid.f90

.. _write:

******
 write
******

.. function:: write(a,geom,VandP,lambda,average)

    Writes the velocity and pressure fields

    :param a: fluid object to write
    :type a: fluid
    :param geom: geometry class to use for write
    :type geom: Optional[body]
    :param VandP: Velocity and Pressure
    :type VandP: Optional[logical]
    :param lambda: lambda2 and vorticity
    :type lambda: Optional[logical]
    :param average: average in z-direction
    :type average: Optional[logical]
    :rtype: N-A

By default this vrites the fluid to a ``fluid.vtp.vtr`` file. If ``geom`` is also passed to the subroutine, it also writes a ``bodyF.vtp.vtr`` file. 

By setting the flag ``VandP.true.`` and ``lambda=.true.``, but velocity and pressure a written, in the standard file format, and a additional file ``vortF.vtp.vtr`` that contains the \lambda_2-criterion and vorticity.

.. code:: fortran

   flow%write(geom,VandP=.true.,lambda=.true.)

If the ``average`` flag is set to ``.true.``, the flow field is averaged in the z-direction, and the results are written to the ``fluid.vtp.vtr`` file.


Relevant files within the repos directory:

* oop/fluid.f90

.. _reset:

**********
 reset u0
**********

.. function:: reset_u0(a)

    resets the velocity stored in a%u0 to a%velocity

    :param a: fluid class to reset
    :type a: fluid
    :rtype: N-A

handy if velocity has been adjusted outside of 'update'

Relevant files within the repos directory:

* oop/fluid.f90