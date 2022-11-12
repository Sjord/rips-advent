# Day 20 - Ornaments

In line 54 the user input `username` is passed to the `userExists()`
method and in line 32-49 it is checked if this user exists in the LDAP
directory. The username is added to the filter string in line 40. The
filter checks if the `simpleSecurityObject` contains an attribute with
the given `uid`. Then the query is sent to the LDAP server in line 43
and if this user exists `true` is returned in line 46. This query would
lead to a Blind LDAP Injection because in line 54 the attacker receives
information if the query was successful or not. To prevent this the
given username is checked in lines 33-38 against a blacklist. This does
not prevent the LDAP Injection but only limits it because attributes
like `userpassword` cannot be read. If you pay attention to the comment
in line 20 there is the attribute `createtimestamp` which is not
included in the blacklist and therefore can be read. With the
`createtimestamp` of the admin user an API token was generated as you
can see in line 10 and with this token it is possible to execute a shell
command in the method `executeCommand()` in line 14. A possible
injection could look like this:

```
?username=admin)(createtimestamp=2*  =&gt; User is found
?username=admin)(createtimestamp=20* =&gt; User is found
```
