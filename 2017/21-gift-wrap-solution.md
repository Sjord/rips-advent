# Day 21 - Gift Wrap

This challenge contains a command injection vulnerability in line 33.
The developer declared `strict_types=1` in line 1 to ensure the the type
hint in the validate function in line 7 throws a `TypeError` exception
if a non-int is passed to the class. Even with strict types enabled
there is an bug with the usage of `array_walk()` which ignores the
strict typing and uses the default weak typing of PHP instead. An
attacker can therefore just append a command to the last parameter that
is executed in the system call. A possible payload could look like
`?p[1]=1&p;[2]=2;%20ls%20-la`.
