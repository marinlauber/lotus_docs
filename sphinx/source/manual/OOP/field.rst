.. _manual-OOP-field:

#######
 Field
#######

This routine defines a continuum mechanical field and basic operators on that field. The fundamental data is a simple 3-D array, but the boundary values, number of spacial dimensions, number of points, and cell location are also included. The fundamental operator is ``point()`` which creates a pointer to a subset of data in the field. Other key operators initiate, copy, apply boundary conditions, and query.

The following shows the different attributes and procedures that are contained in the field module.

.. code:: fortran

    module fieldMod
        !
        ! -- number of ghost cells added outside the domain
        !       note if( QUICK & MPI ) ghosts>1 !!
        integer,parameter,private :: ghosts = 2
        !
        ! -- Field datatype
        type,public :: field
            real,allocatable :: p(:,:,:)     ! data array including ghosts
            integer,private  :: n(3)=1       ! number of points in the array
            integer,private  :: location=0   ! location of data in FV cell
                                             !   0 => cell center (ie. scalar field)
                                             !   1,2,3 => x,y,z face (ie. vector component)
            real    :: bound_val=0.          ! boundary value
            integer :: is,ie,js,je,ks=1,ke=1 ! array bounds
            integer :: ndims=3               ! number of spacial dimensions
            logical :: exit=.false.          ! convection exit
        contains
            procedure,private :: eqField, eqScalar
            generic,public    :: assignment(=) => eqField,eqScalar
            procedure,public  :: init, size => size1, point, applyBC, inner, pos, global, local
            procedure,public  :: filter, max => max1 , mean, perturb, ddn, integral, eval, at
            procedure,public  :: spread => spread1, average, limits, xy_ave, asymmetry, distribute
        end type field
    end module fieldMod

* :ref:`eqField`
* :ref:`eqScalar`
* :ref:`initialise`
* :ref:`size`
* :ref:`point`
* :ref:`applyBC`
* :ref:`inner`
* :ref:`pos`
* :ref:`global`
* :ref:`local`
* :ref:`filter`
* :ref:`max`
* :ref:`mean`
* :ref:`perturb`
* :ref:`ddn`
* :ref:`integral`
* :ref:`eval`
* :ref:`at`
* :ref:`spread`
* :ref:`average`
* :ref:`limits`
* :ref:`xy_ave`
* :ref:`asymetry`
* :ref:`distribute`


.. _eqField:

********
 eqField
********

Sets all attributed of field ``a`` equal to that of field ``b``. 

.. function:: eqField(a, b)

    :param a: field update
    :type a: Class[field]
    :param b: field to equal
    :type b: Class[field]
    :rtype: N-A

This essentially allows you to write something like :

.. code:: fortran

    flow%pressure = flow%sigma

.. note::
    This re-assignes the binary opertaor "=" when aplied to a field type, this is why you are able to write the previous line of code. 

.. _eqScalar:

*********
 eqScalar
*********

Sets value of field to a given scalar value. This is handy to initialise a field with a specific value.

.. function:: eqScalar(a, P)

    :param a: field update
    :type a: Class[field]
    :param P: value to set field to
    :type P: float
    :rtype: N-A

.. code:: fortran

    flow%pressure = 1.0

.. _initialise:

**********
 Initalise
**********

This allocates an array ``p(:,:,:)`` in memory of size ``n(3)+2*ghosts``. The default ghost size is 2. This allows each processor working with his own sub-set of the data to be able to perform finit-difference, filtering, and so on wihtout having to communicate with neighbouring processors.

.. function:: initalise(a, n, P, l, b, exit)

    Initalise a field

    :param a: fluid class to update
    :type a: Class[field]
    :param n: number of point in the field
    :type n: Array[int]
    :param P: boundary value
    :type P: Optional[]
    :param l: field location in cells
    :type l: Optional[]
    :param b: set field to boundary
    :type b: Optional[logical]
    :param exit: exit type
    :type exit: Optional[logical]
    :rtype: N-A 

The default storage location is cell-centred (``l=0.``). This is only valid for scalar-valued fields. 

The array bounds ``a%is``, ``a%ie``, etc. are the indices of the first and last values of the field, not including the halos.

.. note::
    ``a%point()`` and ``a%p(a%is:a%ie, a%js:a%je, a%ks:a%ke)`` represent the same sub-set of the array, the only difference is that the former returns a pointer, and the later an array.


.. _size:

*****
 Size
*****


.. function:: size(a)

    Helper function that return the size of the field, excluding halos.
    
    :param a: field 
    :type a: Class[field]
    :rtype: Array[int]

This is usefull to initialize arrays of the same size

.. code:: fortran

    pressure%init(n=sigma%size())


.. _point:

******
 Point
******

Point to a subsection of the data in the field. By default this point to the field exclusing the halos, see figure below. By specifying a shift, it is very easy to generate finite differnece, filters, and so on.

.. function:: point(a, i, s, arg, all)

    Point to a subsection of the data in the field.
    
    :param a: field 
    :type a: Class[field]
    :param i: 1D shift dircetion
    :type i: Optional[int]
    :param s: 1D shift magnitude
    :type s: Optional[float]
    :param arg: 3D shift
    :type arg: Optional[array(int)]
    :param all: include ghost points or not
    :type all: Optional[logical]
    :rtype: Pointer[real]

They logical argument ``all`` allows to point to the whole array, including halos.

If not given, the default shift magnitude ``s`` is 1. Both positive and negative shift directions can be specified. If a 3D shift is specified, an array containing the shift magnitude must me passed, i.e. ``(\1, 0, 0\)``, which would be equivalent to a shift in the x-direction, with a magnitude of 1 if not explicitley specified (using ``s``).

.. image:: Ghost_cells.png
  :width: 800
  :alt: MPI ghost cell decomposition schematic.

.. warning::
    This function returns a pointer to the subsection of the array that you specified, it can therefore only be assigned to an other pointer, using ``ptr => filed%point()``. 

.. _applyBC:

********
 ApplyBC
********

Apply boundary conditions to the field. A zero derivative condition is set by default. Fields on a cell face (location>0) have the boundary value applied to that face.

.. function:: applyBC(a, dt, b)

    Apply boundary conditions on the field

    :param a: field class on which to apply BC
    :type a: Class[field]
    :param dt: time-step
    :type dt: Optional[float]
    :param b: boundary value
    :type b: Optional[Class[field]]
    :rtype: N-A

This function also ensures that the domain fluxes match the nominal fluxes (if specified), if not, it corrects it.

.. note::
    In addition to handeling the boundary conditions of the problem, this function deals with communicating halos values between processors via the ``myMPI.f90`` module.

.. note::
    Relies on the ``bound(a, i, j, shift)`` function, which returns a pointer that points to the boundary cell ``j``, ghost or adjacent (-1/1), on the ``i`` face. If specified, ``shift`` shifts the pointer.

.. _inner:

******
 Inner
******

.. function:: inner(a, b)

    Take the inner product of two fields excluding ghost values

    :param a: field use for inner product
    :type a: Class[field]
    :param b: second field to use in inner product
    :type b: Optional[field]
    :rtype: Scalar[real]

If a second field ``b`` is specified, the inner-product is take between ``a`` and ``b``, otherwise the inner-product is take with ``a`` and itself. Mathematically, we are doing :

.. math::
   :nowrap:

   \begin{equation}
      c = \langle a, b\rangle = \int a \, b  \, dx \simeq  \sum_{i=1}^{N}a_ib_i
   \end{equation}

.. note::
    This performs a mpi sum.

.. _pos:

****
 Pos
****

.. function:: pos(a, i, j, k)

    Returns the physical position (x,y,z) of cell with indices (i,j,k)

    :param a: field 
    :type a: Optional[field]
    :param i: x-direction index
    :type i: integer
    :param j: y-direction index
    :type j: integer
    :param k: z-direction index
    :type k: integer
    :rtype: Array[real]

The position is adjusted if the field ``a`` is cell- or faced-centered. This is used to interpolate a field at a point, for example.

.. note::
    ``a%pos(1,1,1)`` gives the position of the ghost cell ``a%p(1,1,1)``

.. _global:

******
Global
******

.. function:: global(a, i, coords)

    Convert between local array index to gloabl array index.

    :param a: field class on which to operate 
    :type a: Class[field]
    :param i: local index
    :type i: Array[integer]
    :param coords: MPI coordinate
    :type coords: Optional[Array[integer]]
    :rtype: Array[integer]

Map the local array index to the global array index of the current MPI decomposition. If no MPI decomposition are used (serial simulation), local and global array index are identical. 

``coords`` can also be passed, this can be usefull if you want to know the gloal array index of an index in another MPI part.


.. _local:

******
 Local
******

.. function:: local(a,i,coords)

    convert between globlal to local array index

    :param a: field class on which to operate 
    :type a: Class[field]
    :param i: local index
    :type i: Array[integer]
    :param coords: MPI coordinate
    :type coords: Optional[Array[integer]]
    :rtype: Array[integer]

inverse mapping to ``global``.


.. _filter:

*******
 Filter
*******

.. function:: filter(a)

    :param a: field 
    :type a: Optional[field]
    :rtype: N-A


.. _max:

****
 Max
****

.. function:: max(a)

    :param a: field 
    :type a: Optional[field]
    :rtype: N-A


.. _mean:

*****
 Mean
*****

.. function:: mean(a)

    :param a: field 
    :type a: Optional[field]
    :rtype: N-A


.. _perturb:

********
 Perturb
********

.. function:: perturb(a)

    :param a: field 
    :type a: Optional[field]
    :rtype: N-A


.. _ddn:

****
 ddn
****

.. function:: ddn(a,n,b)

    Take the normal derivative of a field

    :param a: field to take ddn on
    :type a: Class[field]
    :param n: normal field
    :type n: Class[field]
    :param b: resulting field
    :type b: Class[Field]

``n`` is a field (actually a vector) that contains the normal direction coefficient, such that the dot product of this field with the gradient of ``a%p`` gives the normal derivative of the field.

In our case, ``n`` is provided by the body, and its coefficient are calculated once, and updated if needed.

.. warning::
    assuming uniform since the body should be in a uniform grid region


.. _integral:

*********
 Integral
*********

.. function:: integral(a, pow mean)

    Integral of the n-th power of a field

    :param a: field 
    :type a: Class[field]
    :param pow: power
    :type pow: real
    :param mean: compute mean value
    :type mean: Optional[logical]
    :rtype: real

.. math::
   :nowrap:

   \begin{equation}
      c =  \int a^{pow} \, dV \simeq  \sum_{i=1}^{N}a^{pow}_i dv_i
   \end{equation}

.. _eval:

*********
 Evaluate
*********

.. function::  eval(am fn)

    :param a: field 
    :type a: Class[field]
    :type fn: 
    :rtype: N-A


.. _at:

***
 At
***

.. function:: at(a)

    :param a: field 
    :type a: Class[field]
    :rtype: N-A


.. _spread:

*******
 Spread
*******

.. function:: spread(a) 

    :param a: field 
    :type a: Class[field]
    :rtype: N-A


.. _average:

********
 Average
********

.. function:: average(a)

    :param a: field 
    :type a: Class[field]
    :rtype: N-A


.. _limits:

*******
 Limits
*******

This function returns the array indices of the field on which the data is stored, excluding the halos. This is useful to set ``do`` loops over an array, or to operate on its content without having to use pointers.

.. function:: limits(a, is, ie, js, je, ks, ke)

    :param a: field 
    :type a: Class[field]
    :param is: lower limit in i direction
    :type is: int
    :param ie: upper limit in i direction
    :type ie: int
    :param js: lower limit in j direction
    :type js: int
    :param je: upper limit in j direction
    :type je: int 
    :param ks: lower limit in k direction
    :type ks: int
    :param ke: upper limit in k direction
    :type ke: int
    :rtype: N-A


.. _xy_ave:

***********
 XY-average
***********


.. function:: xy_average(a)
    :param a: field 
    :type a: Class[field]
    :rtype: N-A


.. _asymetry:

*********
 Asymetry
*********

.. function:: asymetry(a)

    :param a: field 
    :type a: Class[field]
    :rtype: N-A


.. _distribute:

***********
 Distribute
***********

.. function:: distribute(a)

    :param a: field 
    :type a: class[field]
    :rtype: N-A


Relevant files within the repos directory:

* oop/fluid.f90

