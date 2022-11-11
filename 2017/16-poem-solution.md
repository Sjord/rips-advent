# Day 16 - Poem

This challenge contains two bugs that can be used together to inject
data into the open FTP connection. The first bug is the usage of
`$_REQUEST` in line 9 while only sanitizing `$_GET` and `$_POST` in
lines 14 to 16. `$_REQUEST` is the combination of `$_GET`, `$_POST`, and
`$_COOKIE` but it is only a copy of the values, not a reference.
Therefore the sanitization of `$_GET`, `$_POST`, and `$_COOKIE` alone is
not sufficient. A real world example of a vulnerability that is caused
by a similar confusion can be found <a
href="https://web.archive.org/web/20180519140202/https://blog.ripstech.com/2016/the-state-of-wordpress-security/#all-in-one-wp-security-firewall"
target="_blank">in our blog</a>.  
  
The second bug is the usage of the type-unsafe comparison `==` instead
of `===` in line 25. This enables an attacker to inject and execute new
commands in the existing connection, for example a delete command with
the query string `?mode=1%0a%0dDELETE%20test.file`.
