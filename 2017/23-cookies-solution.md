# Day 23 - Cookies

The `LDAPAuthenticator` class is prone to an LDAP injection in line 24.
By injecting special characters into the username it is possible to
alternate the result set of the LDAP query. Although the `ldap_escape()`
function is used to sanitize the input in lines 19 and 20, a wrong flag
has been passed to the sanitize-calls resulting in
insufficient/incorrect sanitization. Therefore, in this particular
example, the LDAP injection results in an unauthenticated adversary
bypassing the authentication mechanism by injecting the
asterisk-wildcard `*` character as username and password to successfully
login as an arbitrary user.

