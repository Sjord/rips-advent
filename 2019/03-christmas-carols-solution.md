# Day 3 - Christmas Carols

A `temp` parameter is received in line 23 and passed to the
`renderFragment()` function in line 24. Here, the first argument
`fragment` leads to a Code Injection vulnerability in the Velocity
template. In line 16, the fragment (template) is evaluated as Java code
by Velocity. An exploitation is not that easy though. The limitation of
this Template Injection is that the attacker can't execute Java code
directly. However, Java reflection can be used to access interesting
Java classes and to finally execute arbitrary shell commands:

```
user=&temp=#set($s="")#set($stringClass=$s.getClass()
   .forName("java.lang.Runtime").getRuntime()
   .exec("touch hacked.jsp"))$stringClass
```
