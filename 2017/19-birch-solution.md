# Day 19 - Birch

The `ImageViewer` class is prone to remote command execution through the
`size` parameter in line 17. The `preg_replace()` call will purge almost
any non-digit characters. This is not sufficient though because the
function `stripcslashes()` will not only strip slashes but it will also
replace C literal escape sequences with their actual byte
representation. The backslash character is untouched by the
`preg_replace()` call allowing an attacker to inject an octal byte
escape sequence similar to `0\073\163\154\145\145\160\0405\073`. The
`stripcslashes()` function will evaluate this input to `0;sleep 5;`
which is concatenated into the system command and finally executed in
the attackers favor.
