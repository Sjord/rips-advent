# Day 8 - Candle

This challenge contains a code injection vulnerability in line 4. Prior
to PHP 7 the operation `preg_replace()` contained an eval modifier,
short `e`. If the modifier is set, the second parameter (replacement) is
treated as PHP code. We do not have a direct injection point into the
second parameter but we can control the value of `\\1`, as it references
the matched regular expression. It is not possible to escape out of the
`strtolower()` call but since the referenced value is inside of double
quotes, we can use PHP’s curly syntax to inject other function calls. An
attack could look like this: `/?.*={${phpinfo()}}`.
