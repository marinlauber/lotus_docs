.. _manual-update-the-docs:

****************************
 Updating the documentation
****************************

**If you are given help from a more experienced user, you are expected to put that information into the lotus documentation.**

The documentation is built with `Sphinx <http://www.sphinx-doc.org/en/stable/>`_. The content is written in a series of text files in ``lotus_docs/sphinx/source`` in a markdown-like language for simple formatting. 

.. Perhaps the easiest way of updating the documentation is to be added as a colaborator on the main documentation repo. Once this is done you can get your local copy of the documentation with a ``git clone``, add your contribution, and finally push it back with a ``git push orign master``.

Looking at the Sphinx website and the source files should be enough to figure out the basics. The ``sphinx/source`` folder contains the source code, written is reStructured (``.rst``) files. These are the files you want to modify/add your stuff. Once you are happy with your contribution, simply make the documentation into the html format that is needed. This can be done by typing
::
    $ make github

in the ``sphinx`` folder.

.. note::
    ``make github`` makes the html documentation, and copies it from ``sphinx/build`` to ``../docs``, it could also be done by typing ``make html && cp -a build/html/. ../docs``. The reason for this is that github pages looks for html files in ``repo/docs/`` to render the html documentation.

Good luck, and thanks for the help!
