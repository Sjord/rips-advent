# Day 4 - Father Christmas

This code challenge receives the user input via the GET or POST
parameter `url` in line 6. The `url` parameter can be used to exploit an
open redirect vulnerability. The redirect happens in line 10 if `url`
starts with `/`. However, a URI starting with 2 slashes (
`//attacker.org` ) is not a relative URI but an absolute URI without a
scheme. Therefore, the intended check if the URI is relative can be
bypassed with the `url` parameter `//attacker.org`.

Finally, the Location response header looks like:

Location: //attacker.org

and the server redirects the victim to attacker.org.
