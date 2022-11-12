# Day 2 - Eggnog Madness

The parameter `rawJson` in line 20 is user-controlled and parsed as
JSON. The data extracted from the JSON string is then passed to `this`
which invokes the second constructor in line 21. Here, the parameter
`controllerName` and `data` can be misused to instantiate any object and
to control the first parameter of a constructor in line 26. In line 28,
the parameter `task` is used as function name that is executed on the
previously created object. As a result, we can instantiate any object,
control the first argument of the constructor, and we can invoke any
function without parameters.  
For exploitation, an attacker can instantiate a `ProcessBuilder` with a
shell command like `touch hacked.jsp` and then call the `start()`
function which invokes the shell command:
`rawJson={"controller":"java.lang.ProcessBuilder","task":"start","data":["touch","hacked.jsp"]}`  
Note that the "logging" code in line 31 was a feint and is not
vulnerable.
