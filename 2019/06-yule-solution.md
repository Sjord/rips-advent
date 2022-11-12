# Day 6 - Yule

In line 9 untrusted user input is received from the parameter `url`. A
`java.nio.file.Path` instance is created from the given value and the
contents of that file are read by the method
`java.nio.file.Files.readAllBytes()`. This functionality can be used to
read arbitrary files through path traversal. The contents of the file
are not reflected into the response of the request, so it is not
possible to access the content. However, by sending the value
`/dev/urandom` the application can be interrupted because the method
`Files.readAllBytes()` will not terminate until the Java heap is out of
memory. This leads to an infinite file read and finally to a memory
exhaustion (DoS) which is not catched by the `IOException` handler.
