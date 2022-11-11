# Day 24 - Nutcracker

Can you spot the vulnerability?

```php
@$GLOBALS=$GLOBALS{next}=next($GLOBALS{'GLOBALS'})
[$GLOBALS['next']['next']=next($GLOBALS)['GLOBALS']]
[$next['GLOBALS']=next($GLOBALS[GLOBALS]['GLOBALS'])
[$next['next']]][$next['GLOBALS']=next($next['GLOBALS'])]
[$GLOBALS[next]['next']($GLOBALS['next']{'GLOBALS'})]=
next(neXt(${'next'}['next']));
```

[Show Solution](24-nutcracker-solution.md)
