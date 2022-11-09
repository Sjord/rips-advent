# Serendipity 2.0.3: From File Upload to Code Execution

7 Dec 2016 by Hendrik Buchwald

[Serendipity](https://s9y.org/) is an easy to maintain blog engine. There are a lot of
plugins that can be used to extend the functionality, this article will
focus on its core though. With close to 125,000 lines it is a
medium-sized web application. In this post, we will show how attackers
can bypass existing security mechanisms which can lead to remote code
execution attacks.

## RIPS Analysis

The analysis of Serendipity with RIPS took 67 seconds to complete. The
total amount of issues is reasonable for a web application of this size.
Most of the 36 low severe issues detected are information leakage
issues, for example, when an error message leaks the DBMS system of a
corrupted database query. In the following, we will investigate a more
severe issue.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

RIPS identified two critical types of security vulnerabilities in
Serendipity. First, several SQL injections were detected that can be
used to elevate privileges from a regular user to administrative
privileges. We will not explain these issues in detail though because we
already described plenty of SQL injections in our previous advent
calendar gifts. Today, we would like to present something a bit more
unique.

### File Upload Extension Bypass

An interesting vulnerability was revealed in Serendipity \<= 2.0.3
inside the file upload manager. The issue can be exploited by an
attacker after he extended the user privileges. Here, the goal of an
attacker could be to upload an PHP file in order to execute arbitrary
PHP code on the target. However, there is a security system in place
that tries to prevent this.

```php
switch ($serendipity['GET']['adminAction']) {
    case 'add':
⋮
        foreach($_FILES['serendipity']['name']['userfile'] AS $idx => $uploadfiles) {
            foreach($uploadfiles AS $uploadfile) {
                $target_filename = $serendipity['POST']['target_filename'][$idx];
                if (!empty($target_filename)) {
                    $tfile = $target_filename;
                } elseif (!empty($uploadfile)) {
                    $tfile = $uploadfile;
                } else {
                    continue;
                }
                $tfile = serendipity_uploadSecure(basename($tfile));
                if (serendipity_isActiveFile($tfile)) {
                    continue;
                }
```

```php
function serendipity_isActiveFile($file) {
    if (preg_match('@^\.@', $file)) {
        return true;
    }
    if (preg_match('@\.(php.*|[psj]html?|pht|aspx?|cgi|jsp|py|pl)$@i', $file)) {
        return true;
    }
```

The actual file upload is protected by the function
`serendipity_isActiveFile()`. It checks the name of a file and stops the
upload if the file ends with a blacklisted extension, such as `php`. It
is always recommended to specify the allowed file extensions in a
whitelist instead. Although this check hinders attackers depending on
the web server's configuration, there is another way to bypass the
blacklist that works configuration independently.

```php
switch ($serendipity['GET']['adminAction']) {
    case 'add':
⋮
        if ($serendipity['POST']['adminSubAction'] == 'properties') {
            $properties = serendipity_parsePropertyForm();
            break;
        }
```

```php
function serendipity_parsePropertyForm() {
    global $serendipity;
⋮
    foreach($serendipity['POST']['mediaProperties'] AS $id => $media) {
⋮
        if ($serendipity['POST']['oldDir'][$id] != $serendipity['POST']['newDir'][$id]) {
            serendipity_moveMediaDirectory(
                serendipity_uploadSecure($serendipity['POST']['oldDir'][$id]),
                serendipity_uploadSecure($serendipity['POST']['newDir'][$id]),
                'filedir',
                $media['image_id']);
```

```php
function serendipity_moveMediaDirectory($oldDir, $newDir, $type = 'dir', $item_id = null, $file = null) {
    global $serendipity;
    $real_oldDir = $serendipity['serendipityPath'] . $serendipity['uploadPath'] . $oldDir;
    $real_newDir = $serendipity['serendipityPath'] . $serendipity['uploadPath'] . $newDir;
⋮
    if ($type == 'filedir') {
⋮
        $oldfile = $serendipity['serendipityPath'] . $serendipity['uploadPath'] . $oldDir . $pick['name'] . (empty($pick['extension']) ? '' : '.' . $pick['extension']);
        $newfile = $serendipity['serendipityPath'] . $serendipity['uploadPath'] . $newDir . $pick['name'] . (empty($pick['extension']) ? '' : '.' . $pick['extension']);
⋮
        $renameValues = array(array(
            'from'    => $oldfile,
            'to'      => $newfile,
⋮
        ));
        rename($renameValues[0]['from'], $renameValues[0]['to']);
```

With the help of the file manager, it is possible to move files to
another directory. The method `serendipity_moveMediaDirectory()` moves a
file by using PHP's built-in `rename()` function. What is stopping an
attacker from abusing this functionality and from simply renaming an
uploaded file from `something.jpg` to `something.php`? Line 3378 in the
code shown above does. It appends the old file extension to the new name
if there is an extension available, so the file would be renamed to
`something.php.jpg`. There is one thing the developer did not consider
though: a malicious file without extension. An attacker can simply
create a file with the name `php` and move it to the "directory"
`something.`. This elegant solution renames the file to `something.php`
and allows the attacker to execute arbitrary PHP code when accessing the
file in the web root.

## Time Line

| Date | What |
|------|------|
| 2016/09/14 | First contact with vendor |
| 2016/09/15 | Vendor responds with fix |
| 2016/09/26 | [Vendor releases fixed version](https://github.com/s9y/Serendipity/releases/tag/2.0.4) |

## Summary

Serendipity is a solid blog software. There are some rough edges - no
doubt - but its creators are keen on improving the code and making sure
that its users are secure. They responded very fast and a fixed version
was released after only a few days. We would like to thank the
Serendipity team for the very professional collaboration.

The vulnerability shown here is a prime example for the fact that file
uploads in PHP are dangerous. It is easy to make a mistake and the
consequences are disastrous. In order to prevent such vulnerabilities it
is advised to not allow direct access to uploaded files. Instead, these
files can be stored outside of the web directory and meta data can be
retrieved from the database when necessary.
