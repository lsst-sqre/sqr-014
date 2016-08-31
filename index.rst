..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

Introduction
============

Python 2 will `cease to be supported in 2020 <https://www.python.org/dev/peps/pep-0373/>`_ and there is now `a push from the developer community <https://python3statement.github.io>`_ to drop support for Python 2 following upcoming long term support releases.
This includes Jupyter, Matplotlib and `Astropy <https://github.com/astropy/astropy-APEs/blob/master/APE10.rst>`_.
The `scientific community are also starting to investigate Python 3 features <http://python-3-for-scientists.readthedocs.io>`_ that may help them.
Furthermore, the LSST commissioning and operations timeline indicates that we should be planning to use Python 3 in the deployed system.

In August 2015 it was decided with `RFC-60 <https://jira.lsstcorp.org/browse/RFC-60>`_ that we should migrate the stack code (everything in ``lsst_distrib``) to support both Python 3 and Python 2.
Packages outside of ``lsst_distrib`` have to support Python 3 but have the option of dropping Python 2 by filing an RFC demonstrating that they have no external users that will be affected.
This document provides instructions on setting up a Python 3 environment and provides guidance on how to modify code to support both Python 2 and Python 3.

A `preliminary guide on porting to Python 3 <http://dx.doi.org/10.5281/zenodo.60566>`_ was presented at the `LSST 2016 Project and Community Workshop <https://project.lsst.org/meetings/lsst2016/>`_.
This document supercedes that content.

Enabling Python 3
=================

At the time of writing Python 3 porters must use an `lsstsw <https://developer.lsst.io/build-ci/lsstsw.html>`_ environment to do the development.
A new environment must be created and should not be shared with a Python 2 stack installation.
In particular, do not share an EUPS database across both stacks.
This is because the Python binary interfaces are not compatible between 2 and 3, and also EUPS currently has a bug resulting in database corruption when Python 2 attempts to read a database that was updated with Python 3.

.. warning ::
    When sourcing the ``lsstsw`` ``setup.sh`` script, check that no EUPS environment variables
    have been defined. EUPS will not overwrite these variables when doing the setup, leading to
    possible confusion (and data corruption) over which EUPS database is active.

Once the ``lsstsw`` git repository has been cloned, use ``bin/deploy -3`` to install a Python 3 ``miniconda``.
You can then use ``lsstsw`` in the standard way.
For example ``rebuild afw`` will build :lmod:`~lsst.afw` and all its dependencies, and ``rebuild -r tickets/DM-nnnn afw`` will build it using the specified ticket branch.

Build System Caveats
--------------------

All LSST Data Management packages that follow the standard template and are integrated into the EUPS build system, use `SCons <http://scons.org>`_ for building.
`SCons <http://scons.org>`_ does not yet work natively on Python 3.
This means that ``SConstruct``, ``SConscript`` and :lmod:`sconsUtils` ``.cfg`` files should not assume that the Python running the SCons code is the same Python as that being used to run the stack code.
This means that decisions should not be made by querying :mod:`distutils.sysconfig` or :data:`sys.executable` within the code.
Instead, the ``python`` on ``$PATH`` must be queried instead, using :mod:`subprocess`.

.. warning ::
    At this time, to build a Python 3 stack, a ``python2.7`` binary must be available on the system and must be in the ``$PATH``.
    ``python2.6`` is not suitable.

Porting a Package to Python 3
=============================

LSST DM have decided to use the `Python future <http://python-future.org>`_ package to enable Python 2 and Python 3 to coexist.
``future`` was chosen to allow code to be written in modern Python 3 idioms even when running in Python 2.
For example, versions of :func:`range` and :func:`zip` can be imported that return iterators rather than lists, and a :class:`str` class is available that acts like a Python 3 unicode-aware :class:`str`.

We recommend that the port to Python 3 proceed in a number of distinct steps, and code changes should be committed at the end of each of these.

Code cleanup
------------

Many of the Python files will be modified as part of the port and there will be extensive testing during the port itself.
It is therefore an excellent time to run a code cleanup enabled by the `redefinition of the DM Python coding standard in terms of PEP-8 <https://jira.lsstcorp.org/browse/RFC-162>`_.
The `autopep8 <https://github.com/hhatto/autopep8>`_ tool can fix many of the whitespace issues present in the codebase.
Use the command as follows:

.. code-block :: shell

    autopep8 {{package_dir}} --in-place --recursive --ignore E26,E133,E226,E228,N802,N803 --max-line-length 110

where ``{{package_dir}}`` is the directory that is the directory that is being cleaned up and the ignore list specifies which `PEP-8 fixes should not be implemented <https://pycodestyle.readthedocs.io/en/latest/intro.html#error-codes>`_.
This tool will not fix all the style problems with the codebase but it will help a lot with the more basic issues that are common to every file.
Use the `flake8 <http://flake8.readthedocs.io/en/latest/>`_ tool to get a more extensive report on coding standard violations and consider embedding `flake8 <http://flake8.readthedocs.io/en/latest/>`_  support in your favorite editor to get real time feedback on issues.

Modernize
---------

The LSST DM code has been under development since `2004 <http://adsabs.harvard.edu/abs/2004AAS...20510811A>`_ and has a lot of code that was developed when Python was at version 2.3.
This means that there are many places in the code that do not use the ``__future__`` construct to enable true division, print function and absolute import (some code is old enough that is has to enable ``with_statement`` in Python 2.5) for simple Python 3 compatibility.
Also, many exceptions are being caught with ``catch Exception, e`` rather than the more modern ``catch Exception as e``.
All these constructs are not compatible with the current DM coding standard and should be modernized in a distinct commit.

The easiest way to achieve this is to run the ``futurize`` command found in the `Python future <http://python-future.org>`_ package. ``futurize`` has a two stage conversion option where it is possible to first run modernization to Python 2.7 before attempting to support 2.7 and 3.
Run the command as follows:

.. code-block :: shell

    futurize -1 -n -w .

where ``-1`` runs the stage 1 fixer, ``-n`` disables backup files (you are running this in a directory managed by ``git``) and ``-w`` indicates that the files should be updated (default is to tell you what would happen).
Most of the changes are straightforward and should be applied.
The one issue resolves around code that uses the :data:`types.StringType` constant.
This no longer exists on Python 3 and ``futurize`` decides that since it's a synonym for :class:`bytes` that the entire thing should be replaced by :class:`bytes`.
It is probably more likely that the code should be replaced by :class:`str`.

Remove deprecated calls
-----------------------

In Python 3 some :class:`unittest` methods have been officially deprecated and should be replaced with modern equivalents. The :meth:`~unittest.TestCase.assert` method should be replaced with :meth:`~unittest.TestCase.assertTrue` and :meth:`~unittest.TestCase.assertEquals` should be replaced with :meth:`~unittest.TestCase.assertEqual`.
Many of these changes were implemented as part of the `pytest migration <http://dx.doi.org/10.5281/zenodo.60564>`_ (see also `SQR-012 <https://sqr-012.lsst.io>`_) so it is likely that these specific issues will not be encountered.

Futurize
--------

The final step in supporting Python 2 and Python 3 is to run the futurize command to enable the stage 2 conversion.
The following command is recommended:

.. code-block :: shell

    futurize -2 -x division_safe -n -w .

The ``-x`` option here specifically disables the protection of division operators (see below).
``futurize`` will modify imports to enable backwards compatibility shims.
Imports from :mod:`builtins` will be no-ops on Python 3 but on Python 2 will import versions that behave like Python 3 (in Python 3 :mod:`__builtins__` was renamed :mod:`builtins`).
It is common to see imports for ``str``, ``zip``, ``object`` and ``range`` added to the top of a file.
The conversion is not going to be perfect and should not be accepted without examining the changes using an interactive GUI tool or ``git add -p``.
In particular ``futurize`` will make the following changes that should be looked at carefully before applying:

* ``futurize`` is very defensive in its futurizing.
  If a construct is being used in Python 2 that returns a list (such as the :meth:`~dict.keys` method on a :class:`dict`), then ``futurize`` will replace that with ``list(mydict.keys())``.
  Rather than accepting everything, developers should decide whether to accept the :class:`list` variant on a case-by-case basis.
  In a loop it is probable that the :class:`list` should be removed (unless the object is modified in the loop).
  If the list is being returned from a function it should probably be retained.

* In Python 2 the ``/`` division operator can result in integer division depending on the operands, unless ``__future__`` division is enabled.
  In Python 3 division is always "true" division and the ``//`` operator must be used to perform integer division.
  This operator is also available in Python 2 but by default ``futurize`` plays it safe and will replace all divisions with a function call, ``old_div``, that tries to retain the Python 2 default behavior in Python 3.
  It's always best bit to present..
  Unfortunately, disabling this option prevents ``futurize`` from inserting the ``__future__ division`` import at the top of the file (this import is required by our `coding standard <https://developer.lsst.io/coding/python_style_guide.html>`_).

* Methods called ``next()`` that are not implementing Python 2 iterators will confuse the tool.
  It assumes that it is an old-style iterator and will rewrite it using the modern :func:`next` function.
  If ``next()`` methods are present that are not associated with iterators, consider renaming them to avoid confusion in the future.
  In Python 3 (and 2.7) the iterator version should be called ``__next__()``.

* Python 3 :class:`int` is the same as Python 2 :class:`long` and ``futurize`` will convert :func:`long` to :func:`int` during the conversion.
  It may be necessary to retain a Python 2 :class:`long` type in the code and ``future`` allows this via the :mod:`past.builtins` package.
  In many cases in the Data Management code it may be that the distinction between ``int`` and ``long`` is no longer relevant as ``int`` in a 64-bit build of Python is a 64-bit integer and many of the cases in the DM code base use ``long`` to mean "64-bit" rather than arbitrary precision.
  This can be seen in the use of the ``long`` type in :lmod:`lsst.daf.base`.

* ``future`` provides a :class:`str` class that behaves like a Python 3 :class:`str` but is neither a Python 2 :class:`str` nor a Python 2 :class:`unicode` (it is technially a ``newstr``.
  This works fine when a string is being used as a string but unless ``unicode_literals`` is enabled string literals will behave like native :class:`str`.
  This will result in ``type(str) != type("")`` and ``isinstance("", str)`` returning ``False`` on Python 2.
  The correct Python 2 solution for handling ``unicode`` and ``str`` is to use ``isinstance(var, basestring)`` and this is usually the easiest way to write code in Python 2 that can recognize all the string types.
  Python 3 does not understand this class but the future framework can enable this compatibility using ``from past.builtins import basestring``.
  This is the simplest solution to compatibility with the three distinct string classes in Python 2 (when future ``str`` is included) and is one example where Python 2-style code must be used.


Finishing the port
------------------

``futurize`` will not completely fix all of the issues with Python 3/2 compatibility.
Many of the remaining problems will be related to Python 3 being strict when handling strings and bytes.
Python 2 treated bytes and strings as synonyms but one of the key reasons for Python 3 to exist is that it will not use bytes when a string is required.

* Long integer literal (for example ``5L``) syntax is not supported in Python 3 because all integers are long.
  If a long int is required in the Python 2 implementation it must be created using the ``long`` constructor imported from :mod:`past.builtins`.

* :func:`sorted` no longer has a ``cmp`` argument and control of the sort order must be changed to use a key function.

* :meth:`str.translate` changes how it works depending on whether the string is Unicode or bytes.
  Additionally the :func:`string.maketrans` function has been removed from Python 3.
  Use a regular expression substitution instead.

* Some DM code tries to distinguish a string from a sequence by checking if the ``__iter__`` attribute exists.
  This does not work on Python 3 because the ``str`` class does not support ``__iter__``.
  An explicit check for a string instance is required.

* In rare cases a class with multiple inheritance will fail on Python 3 because the two parent classes have metaclasses defined that are incompatible.
  The solution for this is to define a new metaclass that combines both the metaclasses and this may require distinct code paths for Python 2 and 3.
  An example of the technique to use can be found in :lclass:`lsst.daf.persistence.policy.Policy`.

Strings vs Bytes
^^^^^^^^^^^^^^^^

Whilst the change to :func:`print` is always mentioned as the most user-visible change in Python 3, that is not the change that will cause the most pain during the migration.
The real issue is that Python 3 has a strong distinction between arrays of bytes and arrays of characters.
You cannot mix and match bytes and strings and you cannot try to create strings from bytes that are not compatible with the encoding you are using.
Python 2 does not see any difference between a string of ASCII characters and an array of bytes and has no problem with bad Unicode character encodings.

The important thing to remember is that strings are always encoded to bytes using some form of encoding and whenever characters are read from outside the Python layer or characters are written outside the Python, those characters must be encoded to bytes using a particular representation.
In the LSST DM codebase it is almost certainly true that the default, utf-8, encoding can be used as this is compatible with the ASCII that the code base currently assumes and is the encoding most commonly found in modern files.

The :meth:`str.decode` and :meth:`str.encode` methods are used to decode bytes to characters and encode bytes to characters.
Calling them without arguments is generally sufficient.
There are three common sources of strings in the DM codebase: files on disk, output from a shell command, and SWIG C++.

Files on disk are the most straightforward to handle.
Modern :func:`open` can take an encoding argument but this is not usually required unless you know you will be handling characters that are outside the ASCII character set.
Opening a file in character mode is the default and :func:`print` and :meth:`~file.write` will handle the encoding to bytes.
Binary files, such as pickle files, should always be opened in binary mode (``rb`` or ``wb``) and then bytes will be read or written (and strings will be refused on Python 3).

.. note ::
    It may sometimes be required to add a ``u""`` prefix to string literals to allow code that works correctly on Python 3 to also work on Python 2.
    In particular the new :class:`io.StringIO` class in Python 2, insists on Unicode.

Shell commands also return bytes and the results must be decoded:

.. code-block :: python

    import subprocess
    result = subprocess.check_output("ls").decode()

The final string/bytes interface that is commonly present in DM code is the boundary layer between Python and C++.
We have configured the SWIG interface in Python 2 to accept both :class:`bytes` (synonym for :class:`str`) and :class:`unicode` as C++ ``std::string`` (they are always returned as :class:`str` from Python 2 SWIG layer).
This works well in most cases but some C++ interfaces require bytes and can cause serious issues (including crashes) when those bytes are treated as Unicode by SWIG.
There are two solutions available.
One is to completely disable the Unicode handling in the entire SWIG module (by setting ``SWIG_PYTHON_STRICT_BYTE_CHAR`` in the ``.i`` file).
A better solution is to write a manual interface to the functions in question using ``%extend`` in the SWIG interface file.

The following is an actual example from the :lmod:`afw` source.
The :lmeth:`~lsst.afw.math.Random.getState` method returns bytes and not a string, and the :lmeth:`~lsst.afw.math.Random.setState` method accepts bytes, and on Python 3 will not allow a string to be supplied.

.. code-block :: C++

    %extend lsst::afw::math::Random {

        PyObject * getState() const {
            std::string state = self->getState();
            return PyBytes_FromStringAndSize(state.data(), state.size());
        }

        PyObject * setState(PyObject * state) {
            char * buffer = nullptr;
            Py_ssize_t len = 0;
            if (PyBytes_AsStringAndSize(state, &buffer, &len) == 0) {
                std::string state(buffer, len);
                self->setState(state);
                Py_RETURN_NONE;
            }
            return nullptr;
        }

    }

    %ignore lsst::afw::math::Random::getState;
    %ignore lsst::afw::math::Random::setState;


Testing the Port
================

You can use the `Jenkins system <https://ci.lsst.codes>`_ to submit Python 3 jobs for validating progress.
The job submission page includes an option to specify Python 2 or 3.
You will find that finishing the port will be an iterative process.
Things will work fine on Python 3 and then fail on Python 2.
A fix you implement for a unicode issue will then break Python 3.
Hopefully after a few iterations things will settle down.
At that point a full Jenkins build should be done on Python 2 (including ``lsst_distrib``, ``lsst_ci`` and ``lsst_sims``).
It is extremely important that a full Python 2 test build is completed (including all the demos) before merging Python 3 branches to master.
The test coverage of an individual package will not be 100% and some packages higher up the stack may discover an issue.
The most common problem is a string check not being completely correct.

Resources
=========

This document is not attempting to cover all possible Python 3 changes or portability issues.
The following are some documents that were useful:

* `Porting guide from the Python 2 documentation <https://docs.python.org/2/howto/pyporting.html>`_.
* `Python future documentation <http://python-future.org>`_.
* `Python 3 Porting book <http://python3porting.com>`_ by Lennart Regebro (also available in paperback).
