# e107 2.1.2: SQL Injection through Object Injection

23 Dec 2016 by Hendrik Buchwald

The 23rd gift in our advent calendar presents security issues in
[e107](http://e107.org/), a content management system that is in development
since 2013. Among others, we identified a critical issue that allows any
user to update his permissions and to extract sensitive information from
the database by exploiting a PHP Object Injection vulnerability.

![e107](images/e107.png "e107")

## RIPS Analysis

The e107 CMS consists of 317,356 lines of code and was analyzed in about
2 minutes. Many of the vulnerabilities found by RIPS are exploitable,
despite a few exceptions. The main reason for this is that e107 contains
a lot of [unused code](10-non-exploitable-security-issues.md)
from previous releases and thus not all affected functions are
reachable. Most of the SQL injection vulnerabilities are caused by
missing quotes. The input gets escaped properly, but since it is not
surrounded by quotes this [is not sufficient](17-openconf-multi-step-remote-command-execution.md)
and the SQL query can be modified.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

### PHP Object Injection to Privilege Escalation

e107 suffers from a [PHP Object
Injection](https://blog.ripstech.com/2018/php-object-injection/)
vulnerability, i.e. user input is passed to the function
`unserialize()`. The MD5 sum of the user input is checked in line 354,
but there is no secret involved and thus an attacker can simply
calculate the value himself. The correct approach would have been to use
a keyed-hash message authentication code (HMAC) to prevent modifications
by malicious users that do not possess the secret key.

```php
$new_data = base64_decode($_POST['updated_data']);
if (md5($new_data) != $_POST['updated_key'] || ($userMethods->hasReadonlyField($new_data) !== false)) {
    exit();
}
$changedUserData = unserialize($new_data);
```

Unlike in our [Expression Engine
post](05-expressionengine-code-reuse-attack.md),
an attacker does not have to use *property oriented programming* to
reach sensitive code that can be exploited. The return value of
`unserialize()` is stored in the variable `$changedUserData` that is
then used directly in further sensitive operations.

```php
$changedData['data'] = $changedUserData;
$changedData['WHERE'] = 'user_id='.$inp;
if (FALSE === $sql->update('user', $changedData))
```

```php
function update($tableName, $arg, $debug = FALSE, $log_type = '', $log_remark = '') {
    $table = $this->db_IsLang($tableName);
    $arg = $this->_prepareUpdateArg($tableName, $arg);
    $query = 'UPDATE '.$this->mySQLPrefix.$table.' SET '.$arg;
    $result = $this->mySQLresult = $this->db_Query($query, NULL, 'db_Update');
```

Namely, the data is used in a SQL UPDATE query. This query is build by
using the `_prepareUpdateArg()` method.

```php
private function _prepareUpdateArg($tableName, $arg) {
    ⋮
    foreach ($arg['data'] as $fn => $fv) {
        $new_data .= ($new_data ? ', ' : '');
        $ftype = isset($fieldTypes[$fn]) ? $fieldTypes[$fn] : 'str';
        $new_data .= "{$fn}=".$this->_getFieldValue($fn, $fv, $fieldTypes);
        ⋮
    }
    return $new_data .(isset($arg['WHERE']) ? ' WHERE '. $arg['WHERE'] : '');
```

```php
function _getFieldValue($fieldKey, $fieldValue, &$fieldTypes) {
    $type = isset($fieldTypes[$fieldKey]) ? $fieldTypes[$fieldKey] : $fieldTypes['_DEFAULT'];
    switch ($type) {
        case 'str':
        case 'string':
            return "'".$this->escape($fieldValue, false)."'";
```

All values are escaped by `_getFieldValue()` in line 1088 of the method
`_prepareUpdateArg()`. However, with the help of the object injection,
an attacker is able to specify arbitrary array keys and those are not
sanitized in any way. This allows the attacker to modify the `UPDATE`
query that is performed on the `user` table and to set arbitrary values.
It is possible to change permissions of accounts, set new passwords,
read information from other tables, and to inject JavaScript in the
database for persistent cross-site scripting attacks. In short, the
attacker has full control over the application.

## Time Line

| Date | What |
|------|------|
| 2016/11/18 | First contact with vendor |
| 2016/11/21 | Send details to vendor |
| 2016/11/27 | [Vendor pushes fix for most critical vulnerabilities to repository](https://github.com/e107inc/e107/commit/dd2cebbb3ccc6b9212d64ce0ec4acd23e14c5274) |
| 2016/11/29 | Vendor starts patching the less-severe vulnerabilities |
| 2016/11/29 | Coordination with vendor about release of blog post |
| 2016/12/13 | Rechecked status with vendor about release date |
| 2016/12/23 | Vendor releases [fixed version 2.1.3](https://github.com/e107inc/e107/releases/tag/v2.1.3) |

## Summary

The lesson learned by this vulnerability is that one should never trust
array keys in PHP. Despite, these should be treated with care as any
other variable. The security issue could be detected successfully by
RIPS due to its precise array handling and its comprehensive analysis of
PHP object injection vulnerabilities. We would like to thank the e107
team for the very professional collaboration. They responded fast and
worked hard until the very last minute in order to provide a fixed
version. **We urge all users to update to the latest version.**
