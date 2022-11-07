# eFront 3.6.15: Steal your professors password

3 Dec 2016 by Martin Bednorz

![eFront](images/efront.png "eFront")

Today, we present our analysis results for
[eFront](https://www.efrontlearning.com/), the open-source edition of the thriving e-learning
platform eFrontPro. The platform is used by hundreds of organizations
world-wide and consists of over 700,000 lines of PHP code, rendering
manual security analysis ineffective at best. We will analyze two SQL
injections that can be used to leak sensitive data and demonstrate two
related RIPS features for detection.

## RIPS Analysis

Our SAST tool RIPS analyzed the whole application in only 1m 32s and
uncovered many severe security issues. Most of them are straight-forward
SQL Injections that can be used to extract confidential user data, such
as passwords, private messages, course results, and personal
information. Also, a code evaluation vulnerability was detected but
luckily for the users, the affected user-defined function is not in use.
Still, we added it to our demo report.

## Case Study

### Example 1: SQL Injection and Context Tab

The first example is a SQL Injection vulnerability that can be exploited
by any registered student. A student account is easily attainable due to
the nature of the platform and no other obstacles have to be bypassed.
The following code lines trigger the vulnerability.

```php
$id = $_GET['id'];
eF_getTableData("lessons_timeline_topics_data", "*", "id=" . $id);
```

```php
function eF_getTableData($table, $fields = "*", $where = "", $order = "", $group = "", $limit = "") {
    $sql = prepareGetTableData($table, $fields, $where, $order, $group, $limit);
    $GLOBALS['db']->GetAll($sql);
```

Here, the GET parameter `id` is concatenated directly into the third
parameter `$where` of the `eF_getTableData()` function without any input
sanitization or validation. Then, this parameter is passed to the
`prepareGetTableData()` function in order to built a SQL query.

```php
function prepareGetTableData($table, $fields = "*", $where = "", $order = "", $group = "", $limit = "") {
    ⋮
    $sql = "SELECT ".$fields." FROM ".$table;
    if ($where != "") {
        $sql .= " WHERE ".$where;
    }
```

In `prepareGetTableData()`, the `$where` variable is appended to the SQL
query in line 401, as shown in the code summary above. As a result, the
susceptible SQL query is executed within the `GetAll()` method. An
attacker can inject arbitrary SQL commands into the where clause and
alter the SQL query result, such that sensitive information are
retrieved from the database instead of the intended topics data. For
example, the attacker is able to retrieve the professors' passwords or
to read all private messages. The vulnerability can be patched by
casting the GET parameter `id` to integer: `$id = (int)$_GET['id']`.
This way, only numeric values can be used as `id` in the WHERE clause
and no SQL commands can be injected.

RIPS is able to follow the data flow throughout the function and method
calls and detects the tainted data in the SQL query that is finally
executed. The cool thing is that RIPS is also able to reconstruct the
full SQL query that is built. This eases the verification of a
vulnerability drastically, specifically for large data flow traces. In
the context tab
of RIPS, the fully reconstructed SQL query and the injection point of
tainted user input is presented:

```sql
SELECT * FROM G_DBPREFIXlessons_timeline_topics_data WHERE id=$_GET['id']
```

### Example 2: SQL Injection and Array Analysis

Our second example shows a SQL injection that is very similar to the
first one, which is the case for almost all SQL injections within the
application.

```php
foreach ($_POST as $key => $value) {
    $fields[$key] = $value;
}
⋮
$fields['users_LOGIN'] = $_SESSION['s_login'];
⋮
unset($fields['objectives']);
⋮
eF_getTableData("scorm_data_2004", "total_time,id", "content_ID=" . $fields['content_ID'] . " AND users_LOGIN='" . $fields['users_LOGIN'] . "'");
```

```php
function eF_getTableData($table, $fields = "*", $where = "", $order = "", $group = "", $limit = "") {
    $sql = prepareGetTableData($table, $fields, $where, $order, $group, $limit);
    $GLOBALS['db']->GetAll($sql);
```

Here, all data within the superglobal `$_POST` variable is propagated to
the `$fields` array (lines 10-12). Then, the keys `content_ID` and
`users_LOGIN` are concatenated unsanitized to the SQL query via the
`$where` parameter of the `eF_getTableData()` function.

From the static code analysis point of view, this example is interesting
because it requires a highly precise analysis of data flow through
arrays and its keys. First, the propagation of arbitrary tainted data to
arbitrary array keys in the `foreach` loop has to be detected. Then
again, the tainted data in specific array keys are overwritten
(`users_LOGIN`) or unset (`objectives`), effectively eliminating the
possibility for exploitation for these specific keys. In the end, the
analysis must decide which array keys can be tainted by a malicious
user, and which array keys cannot and would lead to false positive
reports.

## Time Line

| Date | What |
|------|------|
| 2016/09/28 | First contact with vendor |
| 2016/09/28 | [Vendor publicly declares EOL for open-source edition](https://github.com/epignosis/efront_open_source/commit/e170f7f896cfa4fdbb97a1ea23a9acf62ad82b30) |

## Summary

In this post, we covered two SQL injection vulnerabilities. While the
issues seem easy to spot from our code summaries, they were hidden in
700,000 lines of code. With the help of our code analysis solution RIPS
the issues were automatically and precisely detected.

The vendor reacted quickly to our submission and, surprisingly, declared
the end-of-life for the open-source edition. For the plenty of users of
this version there are currently no patches available. Hopefully, the
community will be able to address some of these problems, for example by
introducing prepared statements to the code base. We did not perform an
analysis of the professional edition.
