# Day 14 - Snowman

This class is vulnerable to directory traversal because of mass
assignment. The constructor can be used to set arbitrary class
attributes (line 11). By overwriting the attribute `$id` you gain
control over the first parameter of `file_put_contents()` in line 16.
With the help of `../` it is possible to target arbitrary files on the
system that are writable, for example it can be used to create a PHP
shell in the document root. The values that are send to the class are
incremented in line 11 and thus an integer after the operation is done.
The incrementation happens after the assignment though, so the class
attribute contains the original value of `$count`.  
To avoid this security issue be vary careful when using reflection based
on user input to set variables. It is recommended to implement a
white-list verfication that contains the names of all variables that can
be modified.
