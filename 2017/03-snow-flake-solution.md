# Day 3 - Snow Flake

In this code are two security bugs. A file inclusion vulnerability is
triggered by the call of `class_exists()` in line 8. Here, the existance
of a user supplied class name is checked. This automatically invokes the
custom autoloader in line 1 in case the class name is unknown which will
try to include unknown classes. An attacker can abuse this file
inclusion by using a path traversal attack. The lookup for the class
name `../../../../etc/passwd` will leak the passwd file. The attack only
works until version 5.3 of PHP.  
But there is a second bug that also works in recent PHP versions. In
line 9, the class name is used for a new object instantiation. The first
argument of its constructor is under the attackers control as well.
Arbitrary constructors of the PHP code base can be called. Even if the
code itself does not contain a vulnerable constructor, PHP's built-in
class `SimpleXMLElement` can be used for an XXE attack that also leads
to the exposure of files. A real world example of this exploit can be
found in our [blog
post](https://web.archive.org/web/20180519140202/https://blog.ripstech.com/2017/shopware-php-object-instantiation-to-blind-xxe/).
