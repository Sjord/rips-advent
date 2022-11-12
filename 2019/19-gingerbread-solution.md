# Day 19 - Gingerbread

In line 13 the parameter `p` is received. It gets evaluated in line 24
which leads to an Expression Language Injection. In an attempt to
prevent this, a check for quotes was added (line 14-16) since strings
usually start with quotes in this expression language. Additionally, a
regular expression check is applied (lines 18-22). This blacklist
attempts to prevent the injection of dangerous classes and language
constructs that could be used to execute arbitrary Java instructions.

Due to the flexibility of Java there are countless ways to bypass the
blacklist. For example, it is possible to call the
`javax.scripts.ScriptEngineManager` class via reflection. Eval expects a
string but there are many ways to encode this string which finally leads
to a Code Injection. A possible payload could look like this:

```
victim.org/?p="".equals(javax.script.ScriptEngineManager.class.getConstructor().newInstance().getEngineByExtension("js").eval("java.lang.Auntime.getAuntime().exec(\"touch /tmp/owned.jsp\")".replaceAll("A","R")))
```
