# Expression Engine 3.4.2: Code Reuse Attack

5 Dec 2016 by Hendrik Buchwald

[Expression
Engine](https://expressionengine.com/) is a popular general purpose content management system
that is used by thousands of individuals, organizations, and companies
around the world. In this post, we will examine a code reuse
vulnerability that leads to remote code execution. This vulnerability
type allows an attacker to partly control the applications logic and to
chain existing code fragements.

## RIPS Analysis

The analysis with RIPS took about 4 minutes. Overall, the code of
Expression Engine seems to be very robust. Still our analysis results
point out some vulnerabilities. RIPS detected mainly possibilities for a
malicious user to embed HTML and JavaScript code via the administration
interface into the appearance of Expression Engine. Since static
analysis cannot reason about the application's logic, RIPS does not
differentiate between cases where a user is able to inject JavaScript
code by design and cases where this is a security vulnerability.

## Case Study

### Example 1: PHP Object Injection

The most serious issue reported is a [PHP Object
Injection](https://blog.ripstech.com/2018/php-object-injection/)
vulnerability. As the name suggests, this vulnerability allows an
attacker to inject objects of arbitrary class type. The issue resides in
the administration control panel but it can be triggered through
cross-site request forgery (CSRF). An attacker can create a malicious
website and trick an administrator into visiting the page which then
fires the attack against his administration panel in the background. The
following code contains the PHP object injection vulnerability.

```php
public function create() {
⋮
    $values = array(
        'name' => ee()->input->get('name'),
        'url'  => ee('CP/URL')->decodeUrl(ee()->input->get('url'))
    );
⋮
    $this->form($vars, $values);
```

```php
public function decodeUrl($url) {
    unserialize(base64_decode($url));
```

Here, the GET parameter `url` is passed to the sensitive *sink*
`unserialize()` without any sanitization. As already explained in our
[Coppermine
post](02-coppermine-second-order-command-execution.md),
older installations of PHP (before 5.5.37, 5.6.x before 5.6.23, and 7.x
before 7.0.8) are vulnerable to arbitrary code execution due to a
[security
issue](https://www.evonide.com/how-we-broke-php-hacked-pornhub-and-earned-20000-dollar/) in the `unserialize()` function itself.

But even with the latest and greatest PHP version, this object injection
vulnerability poses security risks because it can be used to trigger
other vulnerabilities in the code base that would not be reachable
otherwise. An attacker can inject an object and control its class type
and its properties by modifying the serialized string that is used to
re-create the object. For example, the following string invokes an
object of class `Link` when deserialized: `O:4:"Link":{}`

### Example 2: Chain for XSS to RCE

In order to exploit the PHP object injection vulnerability, an attacker
can utilize an attack technique called [property oriented
programming](https://www.owasp.org/index.php/PHP_Object_Injection). It allows the attacker to chain code fragments of
existing classes in the code base and to jump from one method to another
until a sensitive code is reached that triggers another vulnerability.

Our starting point is the magic method `__toString()` that is
automatically called when an object is type-casted into a string. When
we inject an object of the class `Link`, the object is concatenated into
a template and the `__toString()` method of `Link` is automatically
invoked. This is our initial gadget to takeover the control flow of the
application. In the `__toString()` method, the method `render()` is
called.

```php
class Link {
    public function __toString() {
         return $this->render();
```

```php
class Link {
    public function render() {
        $url = $this->filepicker->getUrl();
```

Since not only the class name, but also the properties of the injected
object are within the control of an attacker, the property `filepicker`
can be chosen arbitrarily. This enables to jump to any method named
`getUrl()` in line 37. Placing an object of class `FilePicker` into this
property then calls the method `getUrl()` of this class.

```php
class FilePicker {
    public function getUrl() {
         $qs = array('directories' => $this->directories);
         return $this->url->make(static::CONTROLLER, $qs);
```

From here, we continue our journey through the `url` property. Again,
any object of any class can be injected into this property and defer the
control flow to arbitrary `make()` methods. An interesting `make()`
method is found in the class `InjectionBindingDecorator`.

```php
class InjectionBindingDecorator implements ServiceProvider {
    public function make() {
         $arguments = func_get_args();
         $name = array_shift($arguments);
         if (isset($this->bindings[$name])) {
             $object = $this->bindings[$name];
             if ($object instanceof Closure) {
                 array_unshift($arguments, $this);
                 return call_user_func_array($object, $arguments);
             }
             return $object;
         }
         array_unshift($arguments, $name);
         array_unshift($arguments, $this);
         return call_user_func_array(
             array($this->delegate, 'make'),
             $arguments
         );
```

At first sight, it is possible to execute arbitrary functions by
injecting into the first argument of `call_user_func_array()`. However,
this is prevented in line 72 that ensures that the argument is a
`Closure` which cannot be injected through `unserialize()` and, thus,
line 74 cannot be reached.

Another call to `call_user_func_array()` is found in line 80 though. It
is not possible to directly execute PHP code here, but at least we can
control the class name that is taken from the property `delegate`. Based
on this property, we can call arbitrary `make()` methods again, but no
other interesting `make()` methods were found in the code base.
Interestingly, when a faulty class name is injected, the error page of
Expression Engine is shown and the name of the class is printed without
any sanitization. This at least leads to a cross-site scripting
vulnerability.

Cross-site scripting in Expression Engine can lead to PHP code
evaluation though because it can be abused to enable the PHP parser for
a template and then to write arbitrary code into it. For this purpose,
an attacker can retrieve the secret token used for CSRF protection and
then submit requests in the name of an administrator. You can see a
video demonstration of this step in our [FreePBX
post](01-freepbx-from-cross-site-scripting-to-remote-command-execution.md).

## Time Line

| Date | What |
|------|------|
| 2016/09/14 | First contact with vendor |
| 2016/09/14 | Vendor responds with fix |
| 2016/09/20 | [Vendor releases fixed version](https://docs.expressionengine.com/latest/about/changelog.html#version-3-4-3) |

## Summary

The code of Expression Engine is well written and the architecture
sophisticated. But as demonstrated in this post, one single
vulnerability is enough to endanger your complete system. A simple code
slip can set off an avalanche and lead to arbitrary code execution for
attackers. The team at EllisLab takes security very seriously, promptly
responded to our reporting, and published a fix only 6 days later. Kudos
and thank you for the professional collaboration.

To prevent PHP object injection vulnerabilities, the deserialization of
user-supplied data should be avoided in general. It is much safer to use
similar functions such as `json_encode()`/`json_decode()`. It is also
possible to protect the input with a [keyed-hash message authentication
code](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) (HMAC) in order to prevent unauthorized modifications.
