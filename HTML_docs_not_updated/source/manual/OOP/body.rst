.. _manual-OOP-body:

#######
 Body
#######

Body module

* :ref:`measure`
* :ref:`update`
* :ref:`fillarrays`
* :ref:`map`

.. _measure:

*********
 Measure
*********


.. function:: measure(a, beta, mu1n, ub)

    Measures the body in the current configuration, if not ``a%upToDate``
    
    :param a: body class to measure
    :type a: Class[body]
    :param beta: mu_0
    :type beta: Class[vector]
    :param mu1n: mu_1
    :type mu1n: Class[vector(3)]
    :param ub: body velocity
    :type ub: Class[vector]
    :rtype: N-A


.. _fillarrays:

************
 Fill arrays
************

.. function:: fill_arrays(a, beta, mu1n, ub)

    populates the ``beta``, ``mu1n`` and ``ub`` arrays

    :param a: body class to measure
    :type a: Class[body]
    :param beta: mu_0
    :type beta: Class[vector]
    :param mu1n: mu_1
    :type mu1n: Class[vector(3)]
    :param ub: body velocity
    :type ub: Class[vector]
    :rtype: N-A


.. _update:

*******
 Update
*******

.. function:: update(a,t)

    updates the possition of the body if a mapping is associated to it

    :param a: body to update
    :type a: Class[body]
    :param t: time
    :type t: real
    :rtype: N-A

if called, this sets ``body%upToDate=.false.``, which ensures that the body is measured afterwards.

.. _map:

****
 Map
****

.. function:: map(a,x)

    maps a point ``x`` in the ference configuration to the current configuration. (Deformation gradient tensor)

    :param a: body to update
    :type a: Class[body]
    :param x: position vector
    :type x: Array[real]
    :rtype: Array[real]

This function return the new coordiantes of the point ``x``. In the case where no body is not assocoated with any mapping, the input is returned.
