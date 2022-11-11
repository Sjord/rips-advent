# Day 20 - Stocking

This challenge contains a server-side request forgery vulnerability. It
allows an attacker to perform requests on behalf of the attacked web
server. Thus, servers can be reached that would otherwise be not
reachable for an external attacker. For example, this can be abused to
perform a port scan and to grab banners (e.g., version of the server) on
an internal network that the web server is part of. The exploitable
parts are the usage of `file_get_contents()` with unfiltered user input
in line 14 and the printing of the error message to the user in line 23.
An attacker can request an internal URI like `?img=http://internal:22`
and would get a response such as
`failed to open stream: HTTP request failed! SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2`
if OpenSSH is running. Information like this can be used to prepare
further attacks. Another popular exploit scenario is the retrieval of
sensitive AWS credentials when attacking an AWS cloud instance. Besides
that, `filter_var()` also accepts `file://` URLs, enabling an attacker
to load local files.
