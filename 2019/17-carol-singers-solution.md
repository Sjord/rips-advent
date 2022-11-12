# Day 17 - Carol Singers

The method `loadEnv()` is called in line 39 and processes the cookie
`env` which is user controlled (lines 26-29). In line 30 the method
`setEnv()` is called with two user controlled parameters (key/value).
The `key` variable is split by `.` into an array and the first entry is
checked against a blacklist (line 15) to prevent the injection of
malicious system properties like `java`.

The previously mentioned blacklist check can be bypassed with a payload
like `.java.xxx` because only the first entry of the split key is
validated against the blacklist check in line 15 and an empty string is
accepted (see line 9/10). After the check is passed all empty string or
null values are removed from the list in line 20 and joined together
with `.` again to build the final property. The goal of an attacker is
to set the system property `java.library.path` (line 22) to the value
`/var/myapp/data` because it is possible to upload files there (line 34)
and thus it is vulnerable to a Library Injection. An attacker has to
upload a malicious library called `libDEOBFUSCATION_LIB.so`. The file
requires the prefix `lib` and the suffix `.so`, otherwise
`System.loadLibrary()` in line 44 won't load it.

Proof of Concept:

```
1. Upload file:
curl -v -F 'upload=@/tmp/libDEOBFUSCATION_LIB.so' http://victim.org/

2. Load malicious library:
curl -v --cookie 'env=.java.library.path@/var/myapp/data'
```
