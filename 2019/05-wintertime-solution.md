# Day 5 - Wintertime

In this code challenge the function `toString` iterates over all HTTP
parameters and formats them into HTML representation. In real world
scenarios this can be used for logging purposes. In line 7 the value
`delimiter` is received and is appended in line 16 to the
`StringBuilder` instance after each parameter value. A denial of service
issue may arise due to the internals of `java.util.StringBuilder`. By
default the StringBuilder object is initialized with an array of size
16. Each time a new value is appended, the StringBuilder instance checks
if the data fits into the array. If not, the size of the array is
doubled. In this case there is a large amplification which can cause the
Java heap to run out of memory. By default Apache Tomcat has a 2MB limit
for POST requests and a maximum amount of 10000 parameters. If we submit
a very large value for parameter `delim` (e.g. 1.8 MB) in combination of
an array with multiple (e.g. 10000) HTTP parameters we have a maximum
amplification of the factor ~20000, considering the StringBuilder
internals.
