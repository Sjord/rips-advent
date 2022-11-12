# Day 15 - Mistletoe

This servlet uses the `find` system command and exposes the directories
of the current folder to the user. This leads to an information leak but
an attacker can do even more harm with this code. In line 9/10 the basic
command is build (`find . -type d`) and in line 12-15 the parameter
`options` is appended to the `find` command. Finally,
`java.lang.ProcessBuilder` is called in line 17/18 to execute the
command.

It is important to say that a direct command injection is not possible
because everything after `find` is treated as an argument. However, the
user controls some options/arguments of this command which leads to an
Argument Injection vulnerability. By injecting the `find` parameter
`-exec` an attacker is still able to execute arbitrary system commands
in the end. A payload like `?options=-exec cat /etc/passwd ;` leads to
the final shell command `find . -type d -exec cat /etc/passwd ;`.
