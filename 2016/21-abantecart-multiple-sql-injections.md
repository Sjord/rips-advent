# AbanteCart 1.2.8 - Multiple SQL Injections

21 Dec 2016 by Martin Bednorz

In our 21st advent calendar gift, we cover
[AbanteCart](http://www.abantecart.com/), a very popular e-commerce solution that just turned 5
years old last month. RIPS found multiple SQL injections, PHP object
injections, and the complementary cross-site scriptings so that the more
severe vulnerabilities can be exploited. Interestingly, the AbanteCart
website was defaced just moments before we send out our analysis report
to the development team.

![AbanteCart](images/abantecart.png "AbanteCart")

![AbanteCart
Hacked](images/abantecart_hack_small.png "AbanteCart Hacked")

## RIPS Analysis

The analysis with RIPS of the well over 200,000 lines of code took 4
minutes to complete. The most critical issues were primarily located in
the language manager of the application and could thus be fixed as a
bundle.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

### Example 1: Authenticated SQL Injection

As an example, we detected a SQL injection vulnerability in the language
manager of AbanteCart. The following code lines are affected.

```php
class ControllerPagesLocalisationLanguage extends AController {
public function loadlanguageData() {
    $this->language->fillMissingLanguageEntries(…, $this->request->post['source_language'], …);
```

```php
class ALanguageManager extends Alanguage {
public function fillMissingLanguageEntries($language_id, $source_language_id = 1, $translate_method = '') {
    $tables = $this->_get_language_based_tables();
    foreach ($tables as $table_name) {
        $this->_clone_language_rows($table_name['table_name'], $pkeys, $language_id, $source_language_id, '', $translate_method);
```

```php
class ALanguageManager extends Alanguage {
private function _clone_language_rows($table, $pkeys, $new_language, $from_language = 1, $specific_sql = '', $translate_method = '') {
    $sql = "SELECT " . $keys_str . "\n\t\t\t\tFROM " . $table . "\n\t\t\t\tWHERE language_id = " . $from_language . $specific_sql;
    $this->db->query($sql);
```

The POST parameter `source_language` is passed on in an unsanitized
state to the `fillMissingLanguageEntries()` function in line 225 and is
then directly used in the function call to `_clone_language_rows()` as
the argument `$from_language` in line 556. Here, the parameter
`$from_language` is used in an unquoted and unsanitized way in the SQL
query in line 868 that is executed later. In this case, it would suffice
to cast the variable `$from_language` to integer in order to fix the
vulnerability.

An attacker is able to use [error-based SQL injection
techniques](https://websec.wordpress.com/2011/04/06/blind-sqli-techniques/)
in order to extract data from the database. For example, customer data
or user credentials can be stolen by generating a SQL error message that
includes the desired secrets.

![AbanteCart SQL
Injection](images/abantecart_sqli.png "AbanteCart SQL Injection")

### Example 2: Cross-Site Scripting

In order to exploit the SQL injection described above we require access
to an administration account. Using the cross-site scripting
vulnerability described in the following example it is possible for an
attacker to gain access to such an account and to cause damage to
unsuspecting customers or the shop's reputation.

```php
define('HTTP_ABANTECART', 'http:/' . $_SERVER['HTTP_HOST'] .
    rtrim(rtrim(dirname($_SERVER['PHP_SELF']), 'static_pages'), '/.\'). '/');
⋮
<a href="<?php echo HTTP_ABANTECART; ?>">Go to main page</a>
```

As can be seen in the short code summary above, the variable
`$_SERVER['PHP_SELF']` is used more or less unsanitized in the the
constant definition `HTTP_ABANTECART` and then printed in line 34. The
`rtrim()` functions only trims whitespaces and the string `static_pages`
from the end of the user-controlled request path. Similarly, the
`dirname()` function returns the directory name of the parent of the
given string, rendering it straightforward to circumvent by adding a
child directory to the request path, similar to `index.php/xss/rips/`.

### Example 3: Authenticated SQL Injection

The last example of our case study describes another SQL injection
vulnerability that occurs due to a simple programming negligence.

```php
class ControllerPagesToolBackup extends AController {
public function main() {
    $this->model_tool_backup->createBackupTask('sheduled_backup', $this->request->post);
```

```php
class ModelToolBackup extends Model {
public function createBackupTask($task_name, $data = array()) {
    $table_list = array();
    foreach($data['table_list'] as $table){
        if(!is_string($table)){ continue; } // clean
        $table_list[] = $this->db->escape($table);
    }
    $sql = "SELECT … FROM information_schema.TABLES WHERE … AND TABLE_NAME IN ('" . implode("','", $data['table_list']) . "')\t";
    $this->db->query($sql);
```

As one can see in the code summaries above, the POST array is used in
the function call to `createBackupTask()` as the `$data` parameter. The
array `$data['table_list']` is then traversed in line 35 and each
element is added in a sanitized way to the new `$table_list` array in
line 37. Unfortunately, this array is not used in the resulting SQL
query. Instead, the original and unescaped data in `$data['table_list']`
is inserted into the SQL query in line 39. This can be easily fixed by
using the correctly escaped `$table_list` variable instead of
`$data['table_list']`.

## Time Line

| Date                            | What |
----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 2016/10/21                      |   Website hacked |
| 2016/10/21                      |   First contact with vendor and issue created on [GitHub](https://github.com/abantecart/abantecart-src/issues/695) (without any critical information) |
| 2016/10/21                      |   [Fix #1 provided in 1.2.9 branch](https://github.com/abantecart/abantecart-src/commit/0d796e38c6641554d6574f29acc88bf09e9f4471) |
| 2016/10/22                      |   [Fix #2 provided in 1.2.9 branch](https://github.com/abantecart/abantecart-src/commit/c2404ae8961f4d41f87bf906ce1fcace2eea0a8e) [Fix #3 provided in 1.2.9 branch](https://github.com/abantecart/abantecart-src/commit/2599931c5f30173dd2f55c2bd8c687968ea91390) |
| 2016/12/20                      |   [Vendor released fixed version](http://www.abantecart.com/shopping-cart-news/abantecart-129-released) |

## Summary

Combining multiple seemingly non-critical security issues can lead to
high risk situations for companies and their customers at the same time,
as we already demonstrated in our previous advent calendar posts.
According to a [Facebook
comment](https://www.facebook.com/AbanteCart/posts/1113871428661237),
the defacement was related to a security issue in Joomla. Since a demo
version with administrator privileges of the e-commerce solution was
available, it might as well could have been an entry point for attackers
though. We would like to thank the AbanteCart team for the quick fixes
of our reported issues.
