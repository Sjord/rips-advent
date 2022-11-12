# Day 23 - Ivy

This challenge contains a Format String Injection vulnerability. The
user input `name` is escaped against Cross-Site Scripting (XSS) in line
17 and concatenated into a format string in line 18. Since `calendar` is
an object, several format strings can be used here and our user input
can also contain input format strings without producing an error. A
format string `%s` calls the internal `toString()` function from the
class `java.util.Calendar` which in turn calls `toString()` from the
respective objects the Calendar object contains. In line 12 a
`java.util.SimpleTimeZone` object is created with an unfiltered user
controlled input `id` which is added to the Calendar object (line 15).
Thus the format string in line 18 can contain a XSS payload which is
inserted via the format string `%s` and printed in line 20 which results
in a Reflective XSS vulnerability.

Proof of Concept:

```
http://victim.org/?id=<script>alert(1)</script>&current_time=Thu Jun 18 20:56:02 EDT 2009&name=%shellos
```
