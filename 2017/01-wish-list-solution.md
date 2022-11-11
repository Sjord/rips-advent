# Day 1 - Wish List

The challenge contains an arbitrary file upload vulnerability in line
13. The operation `in_array()` is used in line 12 to check if the file
name is a number. However, it is type-unsafe because the third parameter
is not set to 'true'. Hence, PHP will try to type-cast the file name to
an integer value when comparing it to the array $whitelist (line 8). As
a result it is possible to bypass the whitelist by prepending a value in
the range of 1 and 24 to the file name, for example "5backdoor.php". The
uploaded PHP file then leads to code execution on the web server.
