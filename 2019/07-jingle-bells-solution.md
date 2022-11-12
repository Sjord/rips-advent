# Day 7 - Jingle Bells

In this challenge the `username` parameter in line 25 is user-controlled
and flows into `jGenerator.writeRawValue()` without sanitization. For
example, an attacker could inject the payload
`?username=foo","permission":"all` which results in:

```
{
  "username":"foo",
  "permission":"all",
  "permission":"none"
}
```

If the JSON object is deserialized in method `loadJson()` (line 13), the
user `foo` has escalated his privileges to `all` permissions through a
JSON injection. Also, a successful exploitation depends on the
implementation of `loadJson()`. To successfully exploit this issue, the
method `loadJson()` must deserialize only the first occurrence of every
key such that the duplicate keys are ignored.
