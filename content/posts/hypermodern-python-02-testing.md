--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 2: Testing"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - nox
  - pytest
  - coverage.py
  - black
  - flake8
---

In this second installment of the Hypermodern Python series, I'm going to
discuss how to add unit tests and linting to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 3: Continuous Integration](../hypermodern-python-03-continuous-integration)
- [Chapter 4: Documentation](../hypermodern-python-04-documentation)
- [Chapter 5: Typing](../hypermodern-python-05-typing)

<!--
This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Unit testing with pytest](#unit-testing-with-pytest)
- [Code coverage with coverage.py](#code-coverage-with-coveragepy)
- [Test automation with Nox](#test-automation-with-nox)
- [Reticulating splines](#reticulating-splines)
- [Mocking with pytest-mock](#mocking-with-pytest-mock)

<!-- markdown-toc end -->

## Unit testing with pytest

It's never too early to add unit tests to a project. While the
[unittest](https://docs.python.org/3/library/unittest.html) framework is part of
the Python standard library, [pytest](https://docs.pytest.org/en/latest/) has
become somewhat of a *de facto* standard.

Let's add this package as a development dependency, using Poetry's `--dev`
option:

```sh
poetry add --dev pytest
```

Organize tests in a [separate file
hierarchy](http://doc.pytest.org/en/latest/goodpractices.html#tests-outside-application-code)
next to `src`, named `tests`:

```sh
tests
├── __init__.py
└── test_console.py

1 directory, 2 files
```

The file `__init__.py` is empty and serves to declare the test suite as a
package. While this is not strictly necessary, it allows your test suite to
mirror the source layout of the package under test, even when modules in
different parts of the source tree [have the same
name](https://github.com/pytest-dev/pytest/issues/3151). Furthermore, it gives
you the option to import modules from within your tests package, for example a
module containing a [factory](https://factoryboy.readthedocs.io/) of customized
objects to supply to your test cases.

The file `test_console.py` contains a test case for the `console` module,
checking if the program exits with a status code of zero.

```python
# tests/test_console.py
import click.testing

from hypermodern_python import console


def test_main_succeeds():
    runner = click.testing.CliRunner()
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

The `CliRunner` is needed to invoke the command-line interface from within a
test case. Since this is likely to be needed by most test cases in this module,
let's turn it into a *test fixture*. Test fixtures are declared as simple
functions with the `pytest.fixture` decorator. Test cases can use a test fixture
by including a function parameter with the same name as the test fixture.

```python
# tests/test_console.py
import click.testing
import pytest

from hypermodern_python import console


@pytest.fixture
def runner():
    return click.testing.CliRunner()


def test_main_succeeds(runner):
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

Invoke `pytest` to run the test suite:

```python
$ poetry run pytest
============================ test session starts =============================
platform linux -- Python 3.8.0, pytest-5.2.2, py-1.8.0, pluggy-0.13.0
rootdir: /hypermodern-python
collected 1 item

tests/test_console.py .                                                 [100%]

============================= 1 passed in 0.03s ==============================
```

## Code coverage with coverage.py

*Code coverage* is a measure of the degree to which the source code of your
program is executed while running its test suite. The code coverage of Python
programs can be determined using a tool called
[Coverage.py](https://coverage.readthedocs.io/). Install it via the
[pytest-cov](https://pytest-cov.readthedocs.io/en/latest/) plugin, which
integrates Coverage.py with `pytest`:

```sh
poetry add --dev pytest-cov
```

Create the `.coveragerc` configuration file to
teach the tool about your source tree layout. The configuration also enables
branch analysis and the display of line numbers for missing coverage.

```ini
# .coveragerc
[paths]
source =
   src
   */site-packages

[run]
branch = true
source = hypermodern_python

[report]
show_missing = true
```

To enable coverage reporting, invoke `pytest` with the `--cov` option:

```python
$ poetry run pytest -- --cov
============================= test session starts ==============================
platform linux -- Python 3.8.0, pytest-5.2.2, py-1.8.0, pluggy-0.13.0
rootdir: /hypermodern-python
plugins: cov-2.8.1
collected 1 item

tests/test_console.py .                                                 [100%]

--------------- coverage: platform linux, python 3.8.0-final-0 -----------------
Name                                 Stmts   Miss Branch BrPart  Cover   Missing
--------------------------------------------------------------------------------
src/hypermodern_python/__init__.py       1      0      0      0   100%
src/hypermodern_python/console.py        6      0      0      0   100%
--------------------------------------------------------------------------------
TOTAL                                    7      0      0      0   100%
============================== 1 passed in 0.09s ===============================
```

The reported code coverage is 100%. This number does not imply that your test
suite has meaningful test cases for all uses and misuses of your program.
However, aiming for 100% code coverage is a good practice, especially for a
fresh codebase. Later, we will see some tools that help you achieve this goal.

## Test automation with Nox

One of my personal favorites, [nox](https://nox.thea.codes/) is a successor to
the venerable [tox](https://tox.readthedocs.io/). At its core, the tool
automates testing in multiple Python environments. It makes it easy to run any
kind of job in an isolated environment, with only those dependencies installed
that the particular job needs.

Install Nox via [pip](https://pip.readthedocs.org/):

```sh
pip install --user --upgrade nox
```

Unlike tox, Nox uses a standard Python file for configuration:

```python
# noxfile.py
import nox


@nox.session(python=["3.8", "3.7"])
def tests(session):
    """Run the test suite."""
    env = {"VIRTUAL_ENV": session.virtualenv.location}
    session.run("poetry", "install", external=True, env=env)
    session.run("pytest", "--cov", *session.posargs)
```

> [This PR](https://github.com/theacodes/nox/pull/245) relieves the need to
> explicitly pass `VIRTUAL_ENV` to Poetry. It has been merged into master and
> will be available with the upcoming Nox release.

This file defines a session named `tests`, which installs the project
dependencies and runs the test suite. Nox will create a virtual environment for
the listed Python versions (3.8 and 3.7), and run the session inside each
environment.

```python
$ nox

nox > Running session tests-3.8
nox > Creating virtual environment (virtualenv) using python3.8 in .nox/tests-3-8
nox > poetry install
...
nox > pytest --cov
...
nox > Session tests-3.8 was successful.
nox > Running session tests-3.7
nox > Creating virtual environment (virtualenv) using python3.7 in .nox/tests-3-7
nox > poetry install
...
nox > pytest --cov
...
nox > Session tests-3.7 was successful.
nox > Ran multiple sessions:
nox > * tests-3.8: success
nox > * tests-3.7: success
```

Nox recreates the virtual environments from scratch at each invocation (a
sensible default). You can speed things up by passing the
`--reuse-existing-virtualenvs (-r)` option:

```sh
nox -r
```

## Reticulating splines

The Hypermodern Python package doesn't do anything useful yet. Let's add a
module for [reticulating
splines](https://www.quora.com/What-does-reticulating-splines-mean/answer/Matt-Sephton?ch=10&share=4d9cf44a&srid=uSlxC):

```python
# src/hypermodern_python/splines.py
import time


def reticulate(count=-1):
    spline = 0
    while count < 0 or spline < count:
        time.sleep(1)
        spline += 1
        yield spline
```

This module defines a single function `splines.reticulate`, which yields an
infinite number of splines, or the number of splines specified by the `count`
parameter. Each spline is reticulated in a time-consuming process.

Let's make this exciting functionality accessible from the command-line. For
users with less than an infinite amount of time on their hands, add an option to
control the number of splines to reticulate.

```python
# src/hypermodern_python/console.py
import click
 
from . import __version__, splines
 
 
@click.command()
@click.option("-n", "--count", default=-1, help="Number of splines to reticulate")
@click.version_option(version=__version__)
def main(count):
    """The hypermodern Python project."""
    for spline in splines.reticulate(count):
        click.echo(f"Reticulating spline {spline}...")
```

Let's see the program in action:

```sh
$ poetry run hypermodern-python -- --count=3

Reticulating spline 1...
Reticulating spline 2...
Reticulating spline 3...
```

## Mocking with pytest-mock

Unfortunately, the test suite now appears to hang. The program reticulates
splines forever and ever when invoked without options. Restrict the number of
splines in the test case to three:

```python
def test_main_succeeds(runner):
    result = runner.invoke(console.main, ["--count=3"])
    assert result.exit_code == 0
```

Now the test suite completes, but it still takes rather long, because each
iteration of `splines.reticulate` invokes `time.sleep` to wait for one second
before yielding a spline. Reticulating splines is a complicated and
time-consuming process. Not a reason to skip these tests altogether!

The [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
standard library allows you to replace parts of your system under test with mock
objects. Add the [pytest-mock](https://github.com/pytest-dev/pytest-mock)
plugin, which integrates the library with `pytest`:

```sh
poetry add --dev pytest-mock
```

The plugin provides a `mocker` fixture which can be used to replace a function
by a mock object. Create a new fixture named `mock_sleep` to mock out the
`time.sleep` function, and use the mock in your test case by adding it as a
function parameter:

```python
# tests/test_console.py
...

@pytest.fixture
def mock_sleep(mocker):
    return mocker.patch("time.sleep")


def test_main_succeeds(runner, mock_sleep):
    ...
```
    
Now the tests pass immediately, without invoking the actual `time.sleep`
function.

While the code coverage is still reported as 100%, the tests do not yet cover
much of the possible behaviour of your program. Before adding more test cases,
move the `mock_sleep` fixture to a separate file named `conftest.py`. Fixtures
in this file are accessible to all tests without requiring an explicit import.

```python
# tests/conftest.py
import pytest


@pytest.fixture
def mock_sleep(mocker):
    return mocker.patch("time.sleep")
```
    
Now add a test file for the `splines` module:

```python
# tests/test_splines.py
from hypermodern_python import splines


def test_reticulate_yields_count_times(mock_sleep):
    iterator = splines.reticulate(2)
    assert sum(1 for _ in iterator) == 2
```

Next, add another test case for the `console` module:

```python
# tests/test_console.py
def test_main_prints_progress_message(runner, mock_sleep):
    result = runner.invoke(console.main, ["--count=1"])
    assert result.output == "Reticulating spline 1...\n"
```

You can also check that a mock object was called by the function under test:

```python
# tests/test_splines.py
def test_reticulate_sleeps(mock_sleep):
    for _ in splines.reticulate():
        break
    assert mock_sleep.called
```

Sometimes, a mock object needs to return something meaningful. Suppose we wanted
to mock out `splines.reticulate` in the `console` test cases. `console.main`
expects a sequence of splines which it can print to the screen. You can
configure the mock object to return a specific value, effectively
short-circuiting the call to the real function. Here's how you do it:

```python
# tests/test_console.py
@pytest.fixture
def mock_splines_reticulate(mocker):
    mock = mocker.patch("hypermodern_python.splines.reticulate")
    mock.return_value = [1, 2, 3]
    return mock


def test_main_succeeds(runner, mock_splines_reticulate):
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

Cheating? Not so. Assuming that `splines.reticulate` is fully covered by other
test cases, mocking it allows us to focus on how `console.main` uses the
function for its own purposes.

<center>[Continue to the next chapter](../hypermodern-python-03-linting)</center>