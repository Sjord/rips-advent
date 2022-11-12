# Day 11 - Carolers

The code challenge extracts a TAR file into a temporary directory. A
user controls the contents of the file `/tmp/uploaded.tar` in line 11. A
TAR file is an archive which holds the contents of various files or
folders. Each file or folder in the archive is mapped to a
`TarArchiveEntry` object. An attacker can control the file name of such
an entry which can be accessed with `TarArchiveEntry.getName()` in line
16. Since the user input reaches the sensitive sink `java.io.File` in
line 16 a path traversal attack (zip slip) can be triggered.

The sanitization in line 16 removes the `../` character sequence from
the name which is insufficient and can be bypassed with the following
payload on a Linux system running a Tomcat server:

```
..././..././..././..././..././var/tomcat/webapps/ROOT/index.jsp
```
