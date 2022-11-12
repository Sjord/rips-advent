# Day 18 - Reindeer

In this snippet we have a Session Fixation vulnerability which leads to
a Command Injection vulnerability. The first thing that happens if you
visit the application is that a new session is created and session
variables are set (lines 43-54). However, an attacker has some control
over the session parameters: the user input `config` is split into
key-value pairs in line 49. After that the processed `config` parameter
is merged with the whitelist on line 51. This leads to full control over
the session variables of the own session. The goal is to reach the
"execute last command" part in lines 23-26 which is only accessible if a
valid password is provided through the authorization header (line 21)
and the session variable `last_command` is set. As mentioned above we
have full control over the session variables but only in the own session
and we do not have a password, so we can not reach it. Luckily, there is
a Session Fixation vulnerability in the lines 35-39 that leads to full
control over a victims cookies. We can send the admin of the app a link
and set his session to our known session via the parameter `config` and
execute our last stored shell command. The authentication header is send
automatically, so the password check is passed.

-   curl "http://victim.org/?config=last\_command@ls" -v
-   Send link to admin and execute the following requests in the
    background:
    -   http://victim.org/?config=JSESSIONID@D4E9132DB9703009B1C932E7C37286ED&save;\_session=yes
    -   http://victim.org/?home=yes
