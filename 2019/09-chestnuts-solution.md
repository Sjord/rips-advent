# Day 9 - Chestnuts

The parameter `whitelist` from line 12 controls a part of the regular
expression pattern in line 21, and the parameter `value` from line 12 is
validated against this expression in line 22. Because we can inject an
arbitrary expression and have control over the value that the expression
is matched against, we can produce heavy CPU consumption with a complex
regular expression (ReDoS). This can lead to a CPU exhaustion and
results in a DoS at least in Java 8.
