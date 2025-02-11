===========
pytest-mock
===========

This plugin provides a ``mocker`` fixture which is a thin-wrapper around the patching API
provided by the `mock package <http://pypi.python.org/pypi/mock>`_:

.. code-block:: python

    import os

    class UnixFS:

        @staticmethod
        def rm(filename):
            os.remove(filename)

    def test_unix_fs(mocker):
        mocker.patch('os.remove')
        UnixFS.rm('file')
        os.remove.assert_called_once_with('file')


Besides undoing the mocking automatically after the end of the test, it also provides other
nice utilities such as ``spy`` and ``stub``, and uses pytest introspection when
comparing calls.

|python| |version| |anaconda| |ci| |coverage| |black| |pre-commit|

.. |version| image:: http://img.shields.io/pypi/v/pytest-mock.svg
  :target: https://pypi.python.org/pypi/pytest-mock

.. |anaconda| image:: https://img.shields.io/conda/vn/conda-forge/pytest-mock.svg
    :target: https://anaconda.org/conda-forge/pytest-mock

.. |ci| image:: https://github.com/pytest-dev/pytest-mock/workflows/test/badge.svg
  :target: https://github.com/pytest-dev/pytest-mock/actions

.. |coverage| image:: https://coveralls.io/repos/github/pytest-dev/pytest-mock/badge.svg?branch=master
  :target: https://coveralls.io/github/pytest-dev/pytest-mock?branch=master

.. |python| image:: https://img.shields.io/pypi/pyversions/pytest-mock.svg
  :target: https://pypi.python.org/pypi/pytest-mock/

.. |black| image:: https://img.shields.io/badge/code%20style-black-000000.svg
  :target: https://github.com/ambv/black

.. |pre-commit| image:: https://results.pre-commit.ci/badge/github/pytest-dev/pytest-mock/master.svg
   :target: https://results.pre-commit.ci/latest/github/pytest-dev/pytest-mock/master

`Professionally supported pytest-mock is now available <https://tidelift.com/subscription/pkg/pypi-pytest_mock?utm_source=pypi-pytest-mock&utm_medium=referral&utm_campaign=readme>`_

Usage
=====

The ``mocker`` fixture has the same API as
`mock.patch <https://docs.python.org/3/library/unittest.mock.html#patch>`_,
supporting the same arguments:

.. code-block:: python

    def test_foo(mocker):
        # all valid calls
        mocker.patch('os.remove')
        mocker.patch.object(os, 'listdir', autospec=True)
        mocked_isfile = mocker.patch('os.path.isfile')

The supported methods are:

* `mocker.patch <https://docs.python.org/3/library/unittest.mock.html#patch>`_
* `mocker.patch.object <https://docs.python.org/3/library/unittest.mock.html#patch-object>`_
* `mocker.patch.multiple <https://docs.python.org/3/library/unittest.mock.html#patch-multiple>`_
* `mocker.patch.dict <https://docs.python.org/3/library/unittest.mock.html#patch-dict>`_
* `mocker.stopall <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.patch.stopall>`_
* ``mocker.resetall()``: calls `reset_mock() <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.reset_mock>`_ in all mocked objects up to this point.

Also, as a convenience, these names from the ``mock`` module are accessible directly from ``mocker``:

* `Mock <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock>`_
* `MagicMock <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.MagicMock>`_
* `PropertyMock <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.PropertyMock>`_
* `ANY <https://docs.python.org/3/library/unittest.mock.html#any>`_
* `DEFAULT <https://docs.python.org/3/library/unittest.mock.html#default>`_ *(Version 1.4)*
* `call <https://docs.python.org/3/library/unittest.mock.html#call>`_ *(Version 1.1)*
* `sentinel <https://docs.python.org/3/library/unittest.mock.html#sentinel>`_ *(Version 1.2)*
* `mock_open <https://docs.python.org/3/library/unittest.mock.html#mock-open>`_
* `seal <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.seal>`_ *(Version 3.4)*

It is also possible to use mocking functionality from fixtures of other scopes using
the appropriate mock fixture:

* ``class_mocker``
* ``module_mocker``
* ``package_mocker``
* ``session_mocker``

Type Annotations
----------------

*New in version 3.3.0.*

``pytest-mock`` is fully type annotated, letting users use static type checkers to
test their code.

The ``mocker`` fixture returns ``pytest_mock.MockerFixture`` which can be used
to annotate test functions:

.. code-block:: python

    from pytest_mock import MockerFixture

    def test_foo(mocker: MockerFixture) -> None:
        ...

The type annotations have been checked with ``mypy``, which is the only
type checker supported at the moment; other type-checkers might work
but are not currently tested.

Spy
---

The ``mocker.spy`` object acts exactly like the original method in all cases, except the spy
also tracks function/method calls, return values and exceptions raised.

.. code-block:: python

    def test_spy_method(mocker):
        class Foo(object):
            def bar(self, v):
                return v * 2

        foo = Foo()
        spy = mocker.spy(foo, 'bar')
        assert foo.bar(21) == 42

        spy.assert_called_once_with(21)
        assert spy.spy_return == 42

    def test_spy_function(mocker):
        # mymodule declares `myfunction` which just returns 42
        import mymodule

        spy = mocker.spy(mymodule, "myfunction")
        assert mymodule.myfunction() == 42
        assert spy.call_count == 1
        assert spy.spy_return == 42

The object returned by ``mocker.spy`` is a ``MagicMock`` object, so all standard checking functions
are available (like ``assert_called_once_with`` or ``call_count`` in the examples above).

In addition, spy objects contain two extra attributes:

* ``spy_return``: contains the returned value of the spied function.
* ``spy_exception``: contain the last exception value raised by the spied function/method when
  it was last called, or ``None`` if no exception was raised.

Besides functions and normal methods, ``mocker.spy`` also works for class and static methods.

As of version 3.0.0, ``mocker.spy`` also works with ``async def`` functions.

.. note::

    In versions earlier than ``2.0``, the attributes were called ``return_value`` and
    ``side_effect`` respectively, but due to incompatibilities with ``unittest.mock``
    they had to be renamed (see `#175`_ for details).

    .. _#175: https://github.com/pytest-dev/pytest-mock/issues/175

Stub
----

The stub is a mock object that accepts any arguments and is useful to test callbacks.
It may receive an optional name that is shown in its ``repr``, useful for debugging.

.. code-block:: python

    def test_stub(mocker):
        def foo(on_something):
            on_something('foo', 'bar')

        stub = mocker.stub(name='on_something_stub')

        foo(stub)
        stub.assert_called_once_with('foo', 'bar')


Improved reporting of mock call assertion errors
------------------------------------------------

This plugin monkeypatches the mock library to improve pytest output for failures
of mock call assertions like ``Mock.assert_called_with()`` by hiding internal traceback
entries from the ``mock`` module.

It also adds introspection information on differing call arguments when
calling the helper methods. This features catches `AssertionError` raised in
the method, and uses pytest's own `advanced assertions`_ to return a better
diff::


    mocker = <pytest_mock.MockerFixture object at 0x0381E2D0>

        def test(mocker):
            m = mocker.Mock()
            m('fo')
    >       m.assert_called_once_with('', bar=4)
    E       AssertionError: Expected call: mock('', bar=4)
    E       Actual call: mock('fo')
    E
    E       pytest introspection follows:
    E
    E       Args:
    E       assert ('fo',) == ('',)
    E         At index 0 diff: 'fo' != ''
    E         Use -v to get the full diff
    E       Kwargs:
    E       assert {} == {'bar': 4}
    E         Right contains more items:
    E         {'bar': 4}
    E         Use -v to get the full diff


    test_foo.py:6: AssertionError
    ========================== 1 failed in 0.03 seconds ===========================


This is useful when asserting mock calls with many/nested arguments and trying
to quickly see the difference.

This feature is probably safe, but if you encounter any problems it can be disabled in
your ``pytest.ini`` file:

.. code-block:: ini

    [pytest]
    mock_traceback_monkeypatch = false

Note that this feature is automatically disabled with the ``--tb=native`` option. The underlying
mechanism used to suppress traceback entries from ``mock`` module does not work with that option
anyway plus it generates confusing messages on Python 3.5 due to exception chaining

.. _advanced assertions: http://docs.pytest.org/en/stable/assert.html


Use standalone "mock" package
-----------------------------

*New in version 1.4.0.*

Python 3 users might want to use a newest version of the ``mock`` package as published on PyPI
than the one that comes with the Python distribution.

.. code-block:: ini

    [pytest]
    mock_use_standalone_module = true

This will force the plugin to import ``mock`` instead of the ``unittest.mock`` module bundled with
Python 3.4+. Note that this option is only used in Python 3+, as Python 2 users only have the option
to use the ``mock`` package from PyPI anyway.

Note about usage as context manager
-----------------------------------

Although mocker's API is intentionally the same as ``mock.patch``'s, its use
as context manager and function decorator is **not** supported through the
fixture:

.. code-block:: python

    def test_context_manager(mocker):
        a = A()
        with mocker.patch.object(a, 'doIt', return_value=True, autospec=True):  # DO NOT DO THIS
            assert a.doIt() == True

The purpose of this plugin is to make the use of context managers and
function decorators for mocking unnecessary, so it will emit a warning when used as such.

If you really intend to mock a context manager, ``mocker.patch.context_manager`` exists
which won't issue the above warning.


Install
=======

Install using `pip <http://pip-installer.org/>`_:

.. code-block:: console

    $ pip install pytest-mock

Changelog
=========

Please consult the `changelog page`_.

.. _changelog page: https://github.com/pytest-dev/pytest-mock/blob/master/CHANGELOG.rst

Why bother with a plugin?
=========================

There are a number of different ``patch`` usages in the standard ``mock`` API,
but IMHO they don't scale very well when you have more than one or two
patches to apply.

It may lead to an excessive nesting of ``with`` statements, breaking the flow
of the test:

.. code-block:: python

    import mock

    def test_unix_fs():
        with mock.patch('os.remove'):
            UnixFS.rm('file')
            os.remove.assert_called_once_with('file')

            with mock.patch('os.listdir'):
                assert UnixFS.ls('dir') == expected
                # ...

        with mock.patch('shutil.copy'):
            UnixFS.cp('src', 'dst')
            # ...


One can use ``patch`` as a decorator to improve the flow of the test:

.. code-block:: python

    @mock.patch('os.remove')
    @mock.patch('os.listdir')
    @mock.patch('shutil.copy')
    def test_unix_fs(mocked_copy, mocked_listdir, mocked_remove):
        UnixFS.rm('file')
        os.remove.assert_called_once_with('file')

        assert UnixFS.ls('dir') == expected
        # ...

        UnixFS.cp('src', 'dst')
        # ...

But this poses a few disadvantages:

- test functions must receive the mock objects as parameter, even if you don't plan to
  access them directly; also, order depends on the order of the decorated ``patch``
  functions;
- receiving the mocks as parameters doesn't mix nicely with pytest's approach of
  naming fixtures as parameters, or ``pytest.mark.parametrize``;
- you can't easily undo the mocking during the test execution;

An alternative is to use ``contextlib.ExitStack`` to stack the context managers in a single level of indentation
to improve the flow of the test:

.. code-block:: python

    import contextlib
    import mock

    def test_unix_fs():
        with contextlib.ExitStack() as stack:
            stack.enter_context(mock.patch('os.remove'))
            UnixFS.rm('file')
            os.remove.assert_called_once_with('file')

            stack.enter_context(mock.patch('os.listdir'))
            assert UnixFS.ls('dir') == expected
            # ...

            stack.enter_context(mock.patch('shutil.copy'))
            UnixFS.cp('src', 'dst')
            # ...

But this is arguably a little more complex than using ``pytest-mock``.

Contributing
============

Contributions are welcome! After cloning the repository, create a virtual env
and install ``pytest-mock`` in editable mode with ``dev`` extras:

.. code-block:: console

    $ pip install --editable .[dev]
    $ pre-commit install

Tests are run with ``tox``, you can run the baseline environments before submitting a PR:

.. code-block:: console

    $ tox -e py38,linting

Style checks and formatting are done automatically during commit courtesy of
`pre-commit <https://pre-commit.com>`_.

License
=======

Distributed under the terms of the `MIT`_ license.

Security contact information
============================

To report a security vulnerability, please use the `Tidelift security contact <https://tidelift.com/security>`__. Tidelift will coordinate the fix and disclosure.

.. _MIT: https://github.com/pytest-dev/pytest-mock/blob/master/LICENSE
