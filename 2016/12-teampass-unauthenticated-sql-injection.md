# Teampass 2.1.26.8: Unauthenticated SQL Injection

12 Dec 2016 by Martin Bednorz

The next gift in our advent calendar reveals security issues in
[Teampass](http://teampass.net/), a collaborative password manager first published in
late 2011. We detected a critical unauthenticated SQL injection and many
file inclusions which could have led to many leaked passwords and angry
users. The issues were reported and fixed earlier this year.

## RIPS Analysis

RIPS was able to analyze the whole project consisting of \~140,000 lines
of code in only 25 seconds, uncovering a lot of severe security
vulnerabilities. The two main types of issues was SQL injection and file
inclusion. Luckily, most of the SQL injections were found in the
installation / upgrade functionality and [are less
severe](10-non-exploitable-security-issues.md).

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

### Example 1: Authenticated Blind SQL Injection

This vulnerability is an excellent example of how RIPS is able to follow
a complex data-flow and its understanding of PHPs unique features.

```php
parse_str($_SERVER['QUERY_STRING']);
rest_get();
```

The code above is the main culprit of many vulnerabilities within this
application. The function `parse_str()`^[1](#fn:1)^ parses a query
string to variables such that, for example, the argument
`data=12&var=foo` initializes the variables `$data = 12` and
`$var = 'foo'`. Here, the HTTP query string from the variable
`$_SERVER['QUERY_STRING']` is used as an input in line 28 which allows
an attacker to create arbitrary variables for the application on a
global scale. Then, the function `rest_get()` is called that creates a
new object of class `NestedTrees`.

```php
function rest_get() {
    $tree = new Tree\NestedTree\NestedTree(prefix_table("nested_tree"), 'id', 'parent_id', 'title');
    $tree->rebuild();
```

```php
class NestedTree {
    public function __construct($table, $idField, $parentField, $sortField) {
        $this->table = $table;
```

```php
function prefix_table($table) {
    global $pre;
    $safeTable = htmlspecialchars($pre.$table);
    if (!empty($safeTable)) {
        return $safeTable;
```

Here, the `table` attribute is set to the result of the call to
`prefix_table("nested_tree")` (see lines 235 and 33). Looking closely at
the function definition one can see that the global variable `$pre` is
prefixed to the table parameter of the function in line 1180. An
attacker is now able to alter the contents of the global variable `$pre`
and thus the table name via simple GET parameters because of the
`parse_str()` call explained earlier. In addition, the usage of
`htmlspecialchars()` is not sufficient enough to sanitize against SQL
injections, although it would not have mattered in this particular case
because there are no quotes that an attacker needs to break out of. RIPS
detected the insufficient sanitization due to its [context-sensitive
taint
analysis](/web/20201108100400/https://blog.ripstech.com/2016/introducing-the-rips-analysis-engine/).

```php
class NestedTree {
public function rebuild() {
    $this->getTreeWithChildren();
```

```php
class NestedTree {
public function getTreeWithChildren() {
    $query = sprintf('select %s from %s order by %s', join(',', $this->getFields()), $this->table, $this->fields['sort']);
    mysqli_query($link, $query);
```

In the end, the tainted table name is used unsanitized in the query
executed by `mysqli_query()` and can be exploited to read arbitrary
values from the database. Since Teampass is a password manager, there is
very likely sensitive data found in the database.

Interestingly enough, we were not able to exploit the vulnerability
because another SQL injection was triggered every time we changed the
global variable `$pre` via the GET parameters. Luckily for us, the newly
found SQL injection requires no authentication to be exploited and thus
is even more critical. The following example of our case study describes
the vulnerability.

### Example 2: Unauthenticated Blind SQL Injection

The security issue in this example is very similar to the previous one,
only that it is much easier to exploit because there is no
authentication required. The following code lines are affected.

```php
parse_str($_SERVER['QUERY_STRING']);
rest_get();
```

```php
function rest_get () {
    if(apikey_checker($GLOBALS['apikey'])) {
```

```php
function apikey_checker ($apikey_used) {
    teampass_connect();
    $apikey_pool = teampass_get_keys();
```

```php
function teampass_get_keys() {
    global $server, $user, $pass, $database, $link;
    teampass_connect();
    $response = DB::queryOneColumn("value", "select * from ".prefix_table("api")." WHERE type = %s", "key");
```

As previously, the main culprit is the call to the `parse_str()`
function, enabling an attacker to create arbitrary global variables. For
every GET request to the API, the API key has to be checked for
correctness using the `apikey_checker()` function. Here, same as before,
the function `prefix_table()` prefixes the table name `api` using the
global variable `$pre` and later executing the query with the
`queryOneColumn()` method. The attacker is able to alter the `pre`
variable and to select arbitrary data from the database without any
requirements of authentication.

### Example 3: File Inclusion

There is also an interesting file inclusion vulnerability using the same
entry point as we saw in the examples before located in the following
code lines. Many file inclusion operations in the application are
similary affected and thus led to a high amount of valid issue reports.

```php
parse_str($_SERVER['QUERY_STRING']);
rest_get();
```

```php
function rest_get() {
    cryption($data['pw'], SALT, $data['pw_iv'], 'decrypt');
```

```php
function cryption($p1, $p2, $p3, $p4 = null) {
    require_once $_SESSION['settings']['cpassman_dir'] . '/includes/libraries/Encryption/Encryption/Crypto.php';
```

Here, an attacker can set the variable
`$_SESSION['settings']['cpassman_dir']` via GET parameters using the
`parse_str()` function and is thus able to include files unintended by
the developer. Depending on the server configuration, it is possible to
include arbitrary remote files, rendering this vulnerability highly
critical. Luckily, the `require_once` statement is only accessible with
an authenticated user and the constant `DEFUSE_ENCRYPTION` being `true`
which is not the default value for the affected version.

## Rescan of the fixed version


The diagram above depicts a consolidated statistic of the rescan of the
updated version of Teampass (2.1.26.8 2.1.26.9). As described [in our
previous
post](11-rescanning-applications-with-rips.md),
RIPS compares the analysis results of both scans and is able to
precisely identify fixed, old, and new issues inside the updated
application version. As can be seen in this case, the new version fixed
a lot of issues and only introduced one new XSS issue into the code
base. You can click on the labels in the legend (Old, Fixed, New) in
order to alter the display.

## Time Line

| Date | What |
|------|------|
| 2016/06/15 | First contact with vendor, initiated by vendor |
| 2016/06/16 | Exchange of details about vulnerabilities |
| 2016/06/27 | [Vendor releases fixed version](https://github.com/nilsteampassnet/TeamPass/releases/tag/2.1.26.9) ([changelog](https://github.com/nilsteampassnet/TeamPass/commit/caa8920b67b68fcc0f66deeebbb7b3d77730513c#diff-42ba1d994f4fa8c6ad17a7efae7936cc)) |

## Summary

Password managers are very sensitive tools and even more fragile when
they are deployed as a web application. We found many critical issues
due to one PHP built-in function used improperly which is why the
precise understanding of all security-relevant PHP functions is a core
feature in our analysis engine. Looking at the amount of issues fixed by
the vendor, the release time of the updated version was very fast and we
want to thank the Teampass developers for their professional and quick
cooperation!
