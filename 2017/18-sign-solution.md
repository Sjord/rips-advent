# Day 18 - Sign

This challenge contains a bug in the usage of the `openssl_verify()`
function in line 5 that leads to an authentication bypass in line 7. The
function has three return values: `1` if the signature is correct, `0`
if the signature verification failed, and `-1` if there was an error
while performing the verification. So if an attacker generates a valid
signature for the data using another algorithm than the one
`pub_key.pem` is using, the `openssl_verify()` function returns `-1`
which is casted to `true` automatically. To avoid this problem use the
type-safe comparison `===` to validate the return value of
`openssl_verify()`, or consider using a different
[library for cryptography](https://paragonie.com/blog/2015/11/choosing-right-cryptography-library-for-your-php-project-guide).
