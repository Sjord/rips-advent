# PHPKit 1.6.6: Code Execution for Privileged Users

8 Dec 2016 by Martin Bednorz

Today's gift in our advent calendar contains
[PHPKit](http://www.phpkit.com/), a German web content management system in development
since early 2002. With its \~42,000 lines of code it is a rather small
application and the latest version is 1.6.6. This post describes two
severe vulnerabilities in the administration section that require a
minimal user permission for exploitation.

## RIPS Analysis

Within only 24 seconds, the analysis with RIPS completed and uncovered
critical security vulnerabilities, mainly in the administration section
of the application. As we demonstrated in multiple previous calendar
posts, these vulnerabilities can be chained with other vulnerabilities
that first escalate to administrative privileges and then allow to
exploit these issues.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post since there are no fixes available.

## Case Study

### Example 1: File Upload

The first vulnerability enables a malicious user to upload arbitrary
files - including a PHP backdoor - that can be stored anywhere on the
web server depending on its file permission configuration. The only
requirement is that the attacker has access to a user account with the
privilege to upload images which could be available, for example, for
editorial staff. The following code lines are affected.

```php
$UPLOAD = new UPLOAD();
$UPLOAD->images($_FILES['image_file'], '../' . $config['image_archive'], $_POST['image_name']);
```

```php
class UPLOAD {
public function images($file = '', $dir = '.', $filename = '') {
    $filename = $filereturn[1] = $dir . '/' . $filename;
    move_uploaded_file($file['tmp_name'], $filereturn[1]);
```

Here, the POST parameter `image_name` is used completely unsanitized in
the sensitive function `move_uploaded_file()`. This allows an attacker
to upload arbitrary files into the web root and thus, to execute custom
PHP code on the targeted server. There is no file extension check as
described and bypassed in our \[previous calendar
post\](/2016/serendipity-from-file-upload-to-code-execution/. It is
advised to not allow users to upload files into the web root and that
the file name is not in full control of the uploading user.

### Example 2: SQL Injection

The SQL injection in this example is rather simple and multiple similar
issues exist across the application. The following lines of code are
affected.

```php
$select_navcat = $_POST['navigation_cat'];
$select_navcat = $_POST['select_navcat'];
unset($select_navcat);
$select_navcat = 'new';
⋮
$SQL->query('UPDATE ' . pkSQLTAB_NAVIGATION . ' SET navigation_cat=\'' . $_POST['delete_links'] . '\' WHERE navigation_cat=\'' . $select_navcat . '\'');
```

```php
class pkSql {
public function query($querystring = '') {
    mysql_query($querystring, $this->servercon);
```

As shown in the code summaries above, the POST parameter `delete_links`
is not sanitized and used directly in the MySQL query in line 46. An
attacker can easily break out of the quotes and alter the query by
injecting SQL commands. This can be used to change arbitrary columns of
the table or to extract user data. Since RIPS has to reconstruct the SQL
query completely in order to evaluate the injection point, RIPS is also
able to evaluate into which table's columns an attacker is able to write
a payload. This can then be used to detect second-order vulnerabilities,
for example persistent XSS. The SQL injection vulnerability can be
prevented by sanitizing the tainted variable `$_POST['delete_links']`
using the function `mysql_real_escape_string()` to hinder the attacker
from breaking out of the quotes.

## Time Line

| Date | What |
|------|------|
| 2016/09/20 | First contact with vendor |
| 2016/09/20 | Vendor discontinued development (no fixes are going to be published for the time being) |

## Summary

RIPS was able to find many critical security vulnerabilities in a matter
of seconds within the application. Even though the issues are located in
the administration section of the application, the code execution could
be exploited by editorial staff that should not have full access to the
server or an attacker with escalated privileges. Unfortunately, the
development is currently on hold so that no fixes are going to be made
available. Hence, no security issues that help in escalating privileges
are released. We advice all PHPKit administrators to remove unnecessary
upload privileges and to harden the web server's file permissions.
