.. _manual-simulations-serial-simulations:

**********************
 Serial Simulations
**********************

Running serial simulations in Lotus is prety easy, you probably have already done tht if you followed this documentation. Here we will go through a set-up file (``lotuf.f90``) that are where the user defines the simulation he wants to undertake.

The first lines of the simulation contain the name of your program, and imports all the required extrenal modules. 

.. code:: fortran

    program sphere_flow
        use bodyMod,    only: body
        use fluidMod,   only: fluid
        use mympiMod,   only: init_mympi,mympi_end,mympi_rank
        use gridMod,    only: xg
        use geom_shape, only: sphere,pi
        use imageMod,   only: display
        implicit none

Here we import the ``body`` object from the ``bodyMod``, etc. In this case, we only import this instance from the module with the keyword ``only:``, this means that only ``body`` will be availiable in our simulations, no other atribute/method of the class. The same is valid for the other imports.

.. note::
    The ``implicit none`` statement is used to inhibit a very old feature of Fortran that by default treats all variables that start with the letters ``i, j, k, l, m, n`` as integers and all other variables as real arguments.

The next few lines define parameters for the simulations. This is relatively self-explanatory. 

.. code:: fortran

        real,parameter :: D = 2*32, Re = 30e3 ! diameter and Reynolds number
        integer :: n(3) = 32*(/4,2,2/)        ! numer of points
        integer :: b(3) = (/1,1,1/)           ! MPI blocks (product must equal n_procs)
        logical :: root                       ! root processor
        type(fluid) :: flow
        type(body) :: geom

.. note::
    In fortran, ``parameter`` are constant values that are passed to the compiler and replace with their assigned value everywhere they appear in the code. This is very handy to hold constants, as they cannot be changed once assigned a value.
        
.. code:: fortran

        call init_mympi(3,set_blocks=b)
        root = mympi_rank()==0
        !
        if(root) print *,'Setting up the grid, body and fluid'
        if(root) print *,'-----------------------------------'

Next the grid is defined. 

.. code:: fortran

        call xg(1)%stretch(n(1), -2*D, -0.5*D, 0.5*D, 5*D, h_min=2., h_max=12.,prnt=root)
        call xg(2)%stretch(n(2), -2*D, -0.5*D, 0.5*D, 2*D, h_min=2.,prnt=root)
        call xg(3)%stretch(n(3), -2*D, -0.5*D, 0.5*D, 2*D, h_min=2.,prnt=root)
        geom = sphere(radius=0.5*D, center=0)

.. code:: fortran

        call flow%init(n/b,geom,V=(/1.,0.,0./),nu=D/Re)
        !
        if(root) print *,'Starting time update loop'
        if(root) print *,'-----------------------------------'
        if(root) print *,' -t- , -dt- '

.. code:: fortran

        do while(flow%time<10*D)
            call flow%update
            write(9,'(f10.4,f8.4,3e16.8)') flow%time/D,flow%dt, &
                2.*geom%pforce(flow%pressure)/(pi*D**2/4.)
            if(mod(flow%time,0.1*D)<flow%dt) then
            if(root) print '(f6.1,",",f6.3)',flow%time/D,flow%dt
            call display(flow%velocity%vorticity_Z(), 'out_vort', lim = 20./D)
            end if
        end do

.. code:: fortran

        if(root) print *,'Loop complete: writing restart files and exiting'
        if(root) print *,'-----------------------------------'
        call flow%write()
        call mympi_end
    end program sphere_flow

