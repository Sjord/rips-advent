# Day 10 - Anticipation

In this code challenge the user input is mapped from the GET or POST
parameter `name` to the function parameter `name` via the annotation
`@RequestParam`. In line 3 the Content-Type of the response is set to
`text/xml`.

If untrusted user input flows into a response with the Content-Type
`text/xml` an attacker can inject a script tag with the xml namespace
attribute `"http://www.w3.org/1999/xhtml"`. The browser interprets the
content as JavaScript and executes it. To solve this challenge the
attacker first has to escape the CDATA element. The filter can be
bypassed by inserting a space bewteen `]]`. This space character is
stripped by the filter. For all other spaces that are needed for payload
the attacker can use tabs.

The following payload can be used to exploit this challenge:

```
test] ]><something%3Ascript%09xmlns%3Asomething%3D"http%3A%2F%2Fwww.w3.org%2F1999%2Fxhtml">alert(1)<%2Fsomething%3Ascript><![CDATA[
```
