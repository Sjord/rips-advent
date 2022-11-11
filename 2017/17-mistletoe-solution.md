# Day 17 - Mistletoe

This challenge is supposed to be a fixed version of day 13 but it
introduces new vulnerabilities instead. The author tried to fix the DQL
injection by applying `addslashes()` without `substr()` on the user
name, and by hashing the password in line 13 using `md5()`. Besides the
fact that md5 should not be used to hash passwords and that password
hashes should not be compared this way, the second parameter is set to
true. This returns the hash in binary format. The binary hash can
contain ASCII characters that are interpreted by Doctrine. In this case
an attacker could use the value `128` as the password, resulting in
`v�an���l���q��\` as hash. With the backslash at the end the single
quote gets escaped leading to an injection. A possible payload could be
`?user=%20OR%201=1-&passwd;=128`.  
To avoid DQL injections always use bound parameters for dynamic
conditions. Never try to secure a DQL query with `addslashes()` or
similar functions. Additionally, the password should be stored in a
secure hashing format, for example BCrypt.
