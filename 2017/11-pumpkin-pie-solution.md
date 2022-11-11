# Day 11 - Pumpkin Pie

This challenge contains an PHP object injection vulnerability. In line
13 an attacker is able to pass user input into the `unserialize()`
function by altering his cookie *data*. There are two checks in line 11
and 12 that try to prevent the deserialization of objects. The first
check can be easily circumvented, for example by injecting an object
into an array, leading to a payload string beginning with `a:1:` instead
of `O:`. The second check can be bypassed by abusing PHP's flexible
serialization syntax. It is possible to use the syntax `O:+1:` to bypass
this regex. Finally, this means an attacker can inject an object of
class `Template` into the application. After the serialized form is
deserialized and the Template object is instantiated, its destructor is
called when the script terminates (line 31). Now, the attacker
controlled properties `cacheFile` and `template` of the injected object
are used to write to a file in line 21. Thus, the attacker can create
arbitraries files on the file system, for example a PHP shell in the
document root:
`a:1:{i:0;O:%2b8:"Template":2:{s:9:"cacheFile";s:14:"/var/www/a.php";s:8:"template";s:16:"<?php%20phpinfo();";}}`
