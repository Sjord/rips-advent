# phpBB 2.0.23 - From Variable Tampering to SQL Injection

13 Dec 2016 by Johannes Dahse

In our 12th [advent
calendar](00-apav-advent-of-php-application-vulnerabilities.md)
gift, we would like to cover an exciting SQL injection in
[phpBB2](https://sourceforge.net/projects/phpbb/files/). Although phpBB2 was replaced by its successor phpBB3,
it is still one of the most popular bulletin boards. RIPS detected a
less severe but very beautiful SQL injection vulnerability that bases on
a PHP quirk we will examine in detail in this post.

## RIPS Analysis

The forum phpBB2 consists of only \~50,000 lines of code and RIPS took
only 19 seconds for its in-depth security analysis to complete. It found
various PHP object injection vulnerabilities which are less severe due
to missing gadget chains.
Further, many SQL injections are reported due to insufficient
sanitization with `str_replace()`.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post since there are no fixes available.

## Case Study

### Variable Tampering

Among others, RIPS reported a variable tampering issue in the style
configuration page for administrators. The GET parameter `install_to` is
used as the name of a variable.

#### admin/admin_styles.php

```php
$install_to = isset($HTTP_GET_VARS['install_to']) ? urldecode($HTTP_GET_VARS['install_to']) : $HTTP_POST_VARS['install_to'];
⋮
$template_name = ${$install_to};
```

This issue enables an attacker to assign any variable to the
`$template_name` variable. For example, the request
`admin_styles.php?install_to=rips` will lead to the assignment
`$template_name = $rips;`. Depending on the previously declared
variables and the use of `$template_name` this can lead to other
security vulnerabilities. RIPS automatically analyzes all possible
combinations for exploitation and, as a result, reported a related SQL
injection vulnerability.

### SQL Injection

Now it gets interesting. The `$template_name` variable, which is in the
attacker's control, is used within two loops in order to build a SQL
query starting in line 76 in order to install a new template from
existing data.

```php
$install_to = isset($HTTP_GET_VARS['install_to']) ? urldecode($HTTP_GET_VARS['install_to']) : $HTTP_POST_VARS['install_to'];
$style_name = isset($HTTP_GET_VARS['style']) ? urldecode($HTTP_GET_VARS['style']) : $HTTP_POST_VARS['style'];
⋮
$template_name = ${$install_to};
for($i = 0; $i < count($template_name) && !$found; $i++) {
   if( $template_name[$i]['style_name'] == $style_name ) {
        while(list($key, $val) = each($template_name[$i])) {
            $db_fields[] = $key;
            $db_values[] = str_replace("\'", "''" , $val);
        }
    }
}
$sql = "INSERT INTO " . THEMES_TABLE . " (";
for($i = 0; $i < count($db_fields); $i++) {
    $sql .= $db_fields[$i];
    ⋮
}
$sql .= ") VALUES (";
for($i = 0; $i < count($db_values); $i++) {
    $sql .= "'" . $db_values[$i] . "'";
    ⋮
}
$sql .= ")";
$db->sql_query($sql);
```

Apparently, the [*variable
variable*](http://php.net/manual/language.variables.variable.php)
`$template_name` is expected to point to a nested array with template
data in the first array level, and key/value pairs in the second array
level. When a user-supplied `style` GET parameter matches the
`style_name` of the template in line 77, `$key` and `$val` pairs are
extracted from the template array by using the PHP built-in functions
`list()` and `each()` in line 78. Then, these are stored in two separate
arrays `$db_fields` and `$db_values` such that they can be used later as
SQL field names and as the fields' values in the SQL query. While the
`$db_values` are escaped by a global filter and a call to
`str_replace()` in line 80, the `$db_fields` are embedded unsanitized
into the SQL query. However, the template data is always static and
loaded from a .cfg file, and there are no other previously declared
arrays with user-supplied data available.

But how can this complex code be exploited?

The trick is to point the *variable variable* `$template_name` that is
used in the loops to another array that is controlled by the user: the
superglobal `$_GET` or `$_POST` array. Since an attacker can control all
of their key/value pairs, he is able to inject SQL commands via array
keys that are used unsanitized as database fields in line 86. All that
is left to do is to submit a nested array structure with one
`style_name` key matching to the GET parameter `style`.

    admin_styles.php?install_to=_GET&0[style_name]=rips&0[SQL_INJECTION]=1&style;=rips

## Time Line

The development of phpBB2 is abandoned and no patch is available. Please
use the latest phpBB3 version.

## Summary

In today's calendar post, we described a less severe SQL injection issue
in the administrator section of the phpBB2 forum. However, its
complexity demonstrated clearly the challenges for a static code
analysis tool. Only with the precise handling of *variable variables*,
array structures, and PHP built-in functions such as `each()` and
`list()`, a static code analysis tool can follow complex data flows
beyond a straight-forward source-to-sink tainting and uncover complex
security issues.
