---
layout: post
title: "Version metadata are like charging cables: Versioning your package with setuptools_scm"
date: "August 14th 2021"
categories: [Python, Tutorial]
tags: [Python, Git, GitHub, Packaging]
---


If you have ever forgotten to pack a charging cable or something else for a trip, you probably have noticed that we humans are prone to errors in simple and repetitive routine tasks [^fn1]. Because machines have a much lower error rate at such tasks, I prefer to delegate and automate as many routine tasks as possible. Versioning a package is one of such tasks. If you want your package to live on the Python Package Index (PyPI), you must include a version. It's furthermore probably expected by your users that you provide a `__version__` attribute in your package's namespace.

Talking about error-proneness, the obvious and probably most error-prone way to include your version is to hardcode it both in your package metadata (e.g., in a `setup.py`) and in your package's files, most likely the top level `__init__.py`. Say, I want to update the version of my package [binary4fun](https://pypi.org/project/binary4fun/) to 0.2.0 in a new release. Following the aforementioned method, I need to manually put it in the `setup.py`:

```python
from setuptools import setup

setup(
    name="binary4fun",
    version="0.2.0",
    # ...
)
```

And I need to manually update it in the package's files, for example in `src/binary4fun/__init__.py`:

```python
__version__="0.2.0"
```

It is the most error-prone method because you can not only forget to increase the version but also accidentally increase just one of them. In the first version of the binary4fun package, I put a harcoded string only into the `__init__.py` to at least avoid the latter. Then, I read it in the `setup.py`, which looked like this:

```python
import pathlib

from setuptools import find_packages, setup


def get_version(rel_path: str):
    file_path = pathlib.Path(__file__).parent.absolute() / rel_path
    for line in file_path.read_text().splitlines():
        if line.startswith("__version__"):
            return line.split('"')[1]
    raise RuntimeError("Unable to find version string.")


setup(
    name="binary4fun",
    version=get_version("src/binary4fun/__init__.py"),
    # ...
)
```

And in `scr/binar4fun/__init__.py`:

```python
__version__ = "0.2.0"
```
Note that I did not import my package but just read the content of `scr/binar4fun/__init__.py`. You could, for example, also put it into a dedicated `_version.py` or `version.txt` file, although you may have to [manually add](https://blog.ionelmc.ro/2014/06/25/python-packaging-pitfalls/#forgetting-package-data) the file to a `MANIFEST.in`, depending on its location, its file type and your `setup()` arguments – another spot for some sweet errors.

Other options are listed for example [here](https://hynek.me/articles/packaging-metadata/) or in the [Python Packaging User Guide](https://packaging.python.org/guides/single-sourcing-package-version/) by PyPA (the aforementioned method is their first listed entry). If you insist on keeping the version number in the code, there is also for example `bump2version`, which updates all version strings in a configured file list.

## setuptools_scm takes the stage
All of the aforementioned approaches have one thing in common: you hardcode your version in at least one place in the package. [setuptools_scm](https://github.com/pypa/setuptools_scm), as the name says, retrieves the package version from scm metadata instead of from an explicit declaration. Again, coming back to human errors, if you work with a version control system such as Git and publish releases on GitHub, you're working with scm metadata already anyway. The way recommended by the authors to implement `setuptools_scm` is to add it to your build configuration in `pyproject.toml` and a line to invoke version inference:
```
[build-system]
requires = ["setuptools>=40.8.0", "wheel", "setuptools_scm[toml]>=6.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools_scm]
```

The entry tells build tools that `setuptools_scm` is required to build the package and, therefore, it gets fetched the isolated environment together with the other listed packages. To infer the version, `setuptools_scm` looks at three things in the standard configuration (quoted from their [manual](https://github.com/pypa/setuptools_scm#default-versioning-scheme)):

1. latest tag (with a version number)
2. the distance to this tag (e.g. number of revisions since latest tag)
3. workdir state (e.g. uncommitted changes since latest tag)

You can check the to be inferred version with `git describe --tags` or the inference result with:
```
❯ python -m setuptools_scm
Guessed Version 0.2.0
```

If you want, you can create a release, build, and publish on PyPi in GitHub Actions. Since this post is about `setuptools_scm`, here is my approach to creating the release. I implemented a manual workflow, which needs the release version as input. Check out a complete workflow (test, create release, build, and publish) [here](https://gist.github.com/moritzkoerber/630554b36d3670c54fe98f8bc7262bee).
```
on:
  workflow_dispatch:
    inputs:
      tag:
        required: true
jobs:
  release:
    name: Create release
    runs-on: ubuntu-latest
    needs: testing
    steps:
      - name: Create GitHub release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.event.inputs.tag }}
          release_name: ${{ github.event.inputs.tag }}
```

If you decide to also build in GitHub Actions, be careful to use `fetch-depth: 0` to fetch all branches and tags:
```
- name: Checkout source
  uses: actions/checkout@v2
  with:
    fetch-depth: 0
```

## What about \_\_version\_\_?
The steps above ensure a version for the package's metadata. One option to provide a `__version__` attribute is to first let `setuptools_scm` write the inferred version to a file:
```
[tool.setuptools_scm]
write_to = "src/binary4fun/_version.py"
```

Then, upon build, your `_version.py` file (automatically included since it resides within the package path) will contain the tag version written into the designated file:

```python
# coding: utf-8
# file generated by setuptools_scm
# don't change, don't track in version control
version = '0.2.0'
version_tuple = (0, 2, 0)
```

To provide the `__version__` attribute, you need to read it from the designated location. Hence, put something like this in your `__init__.py`:

```python
try:
    from ._version import version as __version__
    from ._version import version_tuple
except ImportError:
    __version__ = "unknown version"
    version_tuple = (0, 0, "unknown version")
```

## A little less conversation, a little more action, please
If you do not want to have a static version written in your package but also do not want to let go of your `__version__` attribute, there is also an option to retrieve the version from the installed package's metadata at runtime. It's simple, you just read the installed package's metadata, which as we've seen above must include a version if it comes from PyPI, via [importlib.metadata](https://docs.python.org/3/library/importlib.metadata.html). To make this work, put something like this in your package's `__init__.py`:

```python
from importlib.metadata import version, PackageNotFoundError

try:
    __version__ = version("binary4fun")
except PackageNotFoundError:
    __version__ = "unknown version"
```

However, be aware that `importlib.metadata` came in fresh in 3.8 and, thus, you'd have to rely on its backport for prior python versions, which means effectively that you have to deal with a(nother) dependency - another "charging cable" to forget.

[^fn1]: Fitts, P. M. (1951). *Human engineering for an effective air navigation and traffic control system*. National Research Council, Washington, D. C.
