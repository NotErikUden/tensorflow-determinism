# Release Procedure

This is the procedure to follow to finalize a release of
`tensorflow-determinism`.

## 1. Check Version Number

Confirm that the version number in `tfdeterminism/version.py` is correct.

## 2. Run Tests

Run all the tests and make sure they all pass.

```
$cd test
$./all.sh
```

## 3. Create Archive

```
python setup.py sdist
```

## 4. Upload to PyPI

Upload the archive to the Python Package Index.

The following assumes that `$HOME/.pypirc` exists and contains the following:

```
distutils]
index-servers =
    pypi
    testpypi

[pypi]
# Use upload tool's default repository URL
username: <username>

[testpypi]
repository: https://test.pypi.org/legacy/
username: <username>
```

### 4a. Test PyPI Server


```
twine upload --repository testpypi dist/tensorflow-determinism-<version>.tar.gz
```

Review the release online and try installing it.

### 4b. Real PyPI Server

```
twine upload --repository pypi dist/tensorflow-determinism-<version>.tar.gz
```

Again, review the release online and try installing it.

## 5. Create GitHub Release

Finally, on GitHub, create a new release with an appropriate version tag
(e.g. `v0.1.0`).
