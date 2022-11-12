# Day 22 - Fruitcake

The parameter `url` is turned into an `URLConnection` object in line 31
through the `getUrl()` method. This method checks if the URL starts with
`http` (line 13), if it is a valid external URL (line 17), and also
redirects are not followed as you can see in line 10. However, this
challenge allows a single redirect through the `Location` header (lines
33-43). If an attacker controlled URL sends an attacker controlled
location header this can be exploited. There is a second `getUrl()`
check in line 39, but since `http://` only has to occur somewhere in the
string the payload is pretty obvious: `victim.org?url=http://evil.com/`.
The server has to send the HTTP header
`Location: https://localhost/?x=http://google.com`.

In the check in line 39 only the URL after `http` is checked, so
`google.com` in this case. The request is sent to `https://localhost/`
though with the query string `x=http://google.com` (line 40/41). The
response body of the requested URL is printed in line 45. This is a
classic Server-Side Request Forgery (SSRF) vulnerability but if you take
a closer look you also see that it is a File Read vulnerability:
`Location: file:///etc/passwd#http://google.com`
