# Coppermine 1.5.42: Second-Order Command Execution

2 Dec 2016 by Martin Bednorz

The second gift in our advent calendar contains descriptions of
vulnerabilities in
[Coppermine](http://coppermine-gallery.net/), a very popular picture gallery application written in
PHP and in active development since 2003. It consists of \~160,000 lines
of code (medium-sized web application) and is downloaded roughly 1,200
times per week.

## RIPS Analysis

The analysis with RIPS took only 53 seconds to complete and it uncovered
a lot of security vulnerabilities - although most of them require
authentication. Nonetheless, these issues are severe because they can be
combined with other security vulnerabilities that allow an attacker to
access an account and to move on from there. Regardless, not all user
accounts of Coppermine should be able to inject SQL queries or execute
commands on the server.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

In the following, we examine a few selected vulnerabilities found by
RIPS. We prefer to describe more complex and interesting vulnerabilities
over straight-forward exploits.

### Example 1: Second-Order Command Execution

This example contains two second-order command execution vulnerabilities
that work very similar, only differing in the operating system they can
be executed in. Lets have a look at the vulnerable code.

```php
$imgObj = $imgObj->cropImage($superCage->post->getEscaped('clipval'));
```

```php
if ($superCage->env->getMatched('OS', '/win/i')) {
    $imgFile = str_replace("'","\"" , $imgFile);
    $cmd = "\"".str_replace("\\","/", $CONFIG['impath'])."convert\" -quality {$this->quality} {$CONFIG['im_options']} -crop {$new_w}x{$new_h}+{$clip_left}+{$clip_top} ".str_replace("\\","/" ,$imgFile )." ".str_replace("\\","/" ,$imgFile );
    exec ("\"$cmd\"", $output, $retval);
} else {
    $cmd = "{$CONFIG['impath']}convert -quality {$this->quality} {$CONFIG['im_options']} -crop {$new_w}x{$new_h}+{$clip_left}+{$clip_top} $imgFile $imgFile";
    exec($cmd, $output, $retval);
}
```

In both cases, the config value `$CONFIG['im_options']` is used directly
in the sensitive sink `exec()`. If an attacker is able to control this
value, it allows him to execute arbitrary commands on the server by
simply attaching a system command to the previous one using the `;`
character. In the following, the code is shown that handles the
configuration values.

```php
foreach ($config_section_value as $adminDataKey => $adminDataValue) {
    $evaluate_value = $superCage->post->getEscaped($adminDataKey);
    cpg_config_set($adminDataKey, $evaluate_value);
```

```php
function cpg_config_set($name, $value) {
    $sql = "UPDATE {$CONFIG['TABLE_CONFIG']} SET value = '$value' WHERE name = '$name'";
    cpg_db_query($sql);
```

With the help of the `cpg_config_set()` function, an administrator can
store arbitrary configration values in the config table. The values are
correctly escaped in order to prevent SQL injection. During
initialization of Coppermine, the values are then loaded from the
database and propagated to the `$CONFIG` array.

```php
// Retrieve DB stored configuration
$result = cpg_db_query("SELECT name, value FROM {$CONFIG['TABLE_CONFIG']}");
while ( ($row = mysql_fetch_assoc($result)) ) {
    $CONFIG[$row['name']] = $row['value'];
} // while
```

Thus, a malicious administrator can execute arbitrary system commands by
performing two steps (*second-order vulnerability*). First, he alters
the `im_options` configuration value with a payload that is stored
persistently in the database. Second, he uses the picture editor that
will load the payload from the database again and inject it into the
system command.

RIPS was able to follow this complex data flow through the database by
analyzing the writing SQL query `UPDATE` and the reading SQL query
`SELECT`, and then matching table columns detected as taintable. In
order to remedy the issue, the command arguments should be sanitized
with the built-in PHP function `escapeshellarg()`. But how can an
attacker abuse this issue remotely without having administrator access
and is this issue critical to fix?

### Example 2: Non-Authenticated SQL Injection

As we have described in our [FreePBX
post](01-freepbx-from-cross-site-scripting-to-remote-command-execution.md),
cross-site scripting can be used to take over user accounts. Further,
RIPS detected several SQL injection vulnerabilities that can be
exploited by an attacker to directly retrieve the user credentials from
the database.

The SQL injection vulnerability described in the following is difficult
to exploit because it requires a MD5 hash collision. However, it is a
nice example of how the use of insecure hash algorithms can lead to
security problems further down the road. Also it demonstrates why not
all security issues detected by static code analysis are equally severe
(but should be fixed nonetheless). The affected code lies in the forgot
password function of Coppermine.

```php
$CLEAN['email'] = $superCage->post->testEmail('email');
$CLEAN['key'] = $superCage->get->getEscaped('key');
$CLEAN['id'] = $superCage->get->getEscaped('id');
⋮
$sql = "SELECT null FROM {$cpg_udb->sessionstable}
         WHERE session_id = '" . md5($CLEAN['key'] . $CLEAN['id']) . "'";
$result = cpg_db_query($sql);
if (!mysql_num_rows($result)) {
    cpg_die($lang_forgot_passwd_php['forgot_passwd'], $lang_forgot_passwd_php['illegal_session']);
}
⋮
$sql = "SELECT {$cpg_udb->field['username']}, {$cpg_udb->field'email']}
         FROM {$cpg_udb->usertable}
         WHERE {$cpg_udb->field['user_id']} = {$CLEAN'id']}";
cpg_db_query($sql);
```

```php
function cpg_db_query($query, $use_link_id = 0) {
    mysql_query($query, $link_id);
```

Here, the GET parameter `id` is escaped, stored in `$CLEAN['id']`, and
used securely in the first SQL query in line 39. However, it is used
insecurely at the end of the second SQL query in line 47 that does not
use quotes around the value. Hence, the escaping is insufficient because
an attacker does not need to break out of quotes when injecting new SQL
commands. RIPS is able to differentiate between quoted SQL contexts and
to decide whether escaping is applied correctly or insufficiently.

In order to exploit this issue, the difficulty is to trigger the
injection point because the session id from the database has to match
the MD5 hash value of the GET parameter `key` concatenated with the
payload injected into `id`. This is theoretically possible for a
determined attacker because he is in possession of his own session id
and can find a collision for the MD5 hash built with his data ([200
billion hashes per
second](https://gist.github.com/epixoip/a83d38f412b4737e99bbef804a270c40), no size bounds for the paramter in the source code).
MD5 is known to have many issues and one should not count on its
long-time security. In addition, due to [Moore's
law](https://en.wikipedia.org/wiki/Moore's_law) this will get increasingly easier from year to year.
Clearly, though, the exploitation is strongly limited and an attacker
would favor a more straight-forward SQL injection.

As with the previous example, the issue can be fixed by quoting the
tainted data in the resulting SQL query because it is already in an
escaped state.

### Example 3: PHP Object Injection

Additionally, RIPS found quite a few PHP Object
Injection
vulnerabilities which are similar to the one presented in this example,
using tainted variables in the `unserialize()` function without any kind
of validation.

```php
$FAVPICS = @unserialize(@base64_decode($superCage->cookie->getRaw($CONFIG['cookie_name'] . '_fav')));
```

As one can see, the cookie named `$CONFIG['cookie_name'] . '_fav'` is
base64 decoded and then directly used for
[deserializing](http://php.net/unserialize)
data. The full cookie name can be easily retrieved from the browser and
an attacker is able to search for [gadget
chains](https://www.owasp.org/index.php/PHP_Object_Injection) in the application code that can further compromise the
application and the server it is running on. Although RIPS did not
detect gadget chains in the code base for exploitation it is important
to note that older installations of PHP (before 5.5.37, 5.6.x before
5.6.23, and 7.x before 7.0.8) are automatically vulnerable to arbitrary
code execution due to a [security
issue](https://www.evonide.com/how-we-broke-php-hacked-pornhub-and-earned-20000-dollar/) in the `unserialize()` function itself, rendering this
issue very critical for many servers. The deserialization of tainted
data should be avoided in general and it is much safer to use similar
functions such as `json_encode()`/`json_decode()`. We will cover the
exploitation of gadget chains in our upcoming calendar posts in depth.

## Time Line

| Date | What |
|------|------|
| 2016/09/23 | First contact with vendor |
| 2016/09/26 | [Vendor released a fix (1.5.44)](http://forum.coppermine-gallery.net/index.php/topic,78867.0.html) |

## Summary

Cross-site scripting vulnerabilities were the most common issues found
by RIPS (\~500) which we did not review fully. The most critical issues,
as the command execution described in this post, occured due to the fact
that configuration values were stored in the database and then later
were not sanitized before used in sensitive functions. The fact that
such configuration values are commonly not under the control of an
attacker (e.g. stored in local configuration files) likely led to a
trusted use without further security checks.

All in all, the Coppermine team reacted very quickly and published a fix
for the most critical issues only three days after our notification,
which is not common at all. We would like to thank the team for their
great collaboration.
