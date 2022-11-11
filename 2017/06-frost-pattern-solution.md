# Day 6 - Frost Pattern

This challenge contains a file delete vulnerability. The bug causing
this issue is a non-escaped hyphen character (`-`) in the regular
expression that is used in the `preg_replace()` call in line 21. If the
hyphen is not escaped, it is used as a range indicator, leading to a
replacement of any character that is not a-z or an ASCII character in
the range between dot (`46`) and underscore (`95`). Thus dot and slash
can be used for directory traversal and (almost) arbitrary files can be
deleted, for example with the query parameters
`action=delete&data;=../../config.php`.
