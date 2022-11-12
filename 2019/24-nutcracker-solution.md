# Day 24 - Nutcracker

This code challenge contains an Object Injection vulnerability. The user
input is supplied via the `@RequestBody` annotation of the Spring
framework and is mapped to parameter `xml` in line 2. The input is then
parsed into a `org.w3c.dom.Document` instance in line 8. After parsing,
the XPath expression
`//com.rips.demo.web.User[@serialization='custom'][1]` is used to filter
out the first of all `com.rips.demo.web.User` nodes with the attribute
`serialization='custom'` (line 10/12). In line 17 the filtered node is
then transformed back into a string and then deserialized in line 20.

There are 2 classes in the challenge, both implementing the
`Serializable` interface, meaning that objects of these classes can be
serialized. In both classes the default deserialization is overridden
using the `readObject` method in lines 18-24 and 38-42. The method
`defaultReadObject()` in line 19 and line 40 reads the non-transient and
non-static fields of the classes from the stream. In line 41 the
transient field `password` of the class `User` is manually read from the
stream. This introduces a security risk since we can hide another object
(of any type, not only string) within the `User` object which is not
filtered out by the XPath expression. The sink is in line 19 in the
`readObject` method of the `Invoker` class. This gadget allows us to
create an arbitrary object by calling the constructor with a string
array and invoking a user-controlled method of this object without
arguments. With the following payload we can create a `ProcessBuilder`
instance and execute an arbitrary shell command.

```
<com.rips.demo.web.User serialization="custom">
  <com.rips.demo.web.User>
    <default>
      <email>peter@gmail.com</email>
      <name>Peter</name>
    </default>
    <com.rips.demo.web.Invoker serialization="custom">
      <com.rips.demo.web.Invoker>
        <default>
          <a>
            <string>touch</string>
            <string>abc</string>
          </a>
          <c>java.lang.ProcessBuilder</c>
          <m>start</m>
        </default>
      </com.rips.demo.web.Invoker>
    </com.rips.demo.web.Invoker>
  </com.rips.demo.web.User>
</com.rips.demo.web.User>
```
