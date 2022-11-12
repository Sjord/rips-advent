# Day 8 - Icicles

The parameter `icons` from line 8 flows into a `File` object in line 11.
It is validated in line 14, and finally it is concatenated into a file
path in line 22. The check in line 14 tries to prevent a path traversal
vulnerability by comparing the file name to the parameter `icons`. The
method `getName()` turns input like `/../../../foo.txt` into `foo.txt`
and prevents a simple path traversal attack. However, if the input is
only `..` nothing is removed by `getName()` and thus it is possible to
bypass the security check. In combination with the parameter `filename`
we are able to traverse one directory level higher and can download any
file we want there.
