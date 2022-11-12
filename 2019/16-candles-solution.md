# Day 16 - Candles

In this challenge the user input is received via the `@RequestParam`
annotation and is mapped to the parameter name of method `findUsers`
(line 10). The code snippet creates a Hibernate Session using the MySQL
driver and queries `UserEntity` objects from the database with a
user-supplied filter. In Hibernate, single-quotes within a string
literal are escaped using double single quotes. The method
`escapeQuotes` (line 5) seems to correctly apply this escaping. However,
we can escape the HQL context and execute plain MySQL queries by
injecting the following payload:

```
test\' or 1=sleep(1) -- -
```

Although this payload looks like a harmless string literal for HQL, it
will cause MySQL to execute the function `sleep`. This (truncated) query
will be sent to the database:

```
... where FIRST_NAME='test\'' or 1=sleep(5)-- -'
```
