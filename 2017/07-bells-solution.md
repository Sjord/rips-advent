# Day 7 - Bells

This challenge suffers from a connection string injection vulnerability
in line 4. It occurs because of the `parse_str()` call in line 21 that
behaves very similar to register globals. Query parameters from the
referrer are extracted to variables in the current scope, thus we can
control the global variable `$config` inside of `getUser()` in lines 5
to 8. To exploit this vulnerability we can connect to our own MySQL
server and return arbitrary values for username, for example with the
referrer
`http://host/?config[dbhost]=10.0.0.5&config;[dbuser]=root&config;[dbpass]=root&config;[dbname]=malicious&id;=1`.
