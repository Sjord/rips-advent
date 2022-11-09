# Precurio 2.1: Remote Command Execution via Xinha Plugin

9 Dec 2016 by Hendrik Buchwald

Precurio is an Intranet portal that can be used as a calendar, phone
directory, and much more. It is available as an open-source and
commercial solution. We focused our analysis exclusively on the
open-source version and detected several critical vulnerabilities that
can be used to execute PHP code on the target system without any form of
authentication.

## RIPS Analysis

RIPS detected many security vulnerabilities, such as SQL injection and
cross-site scripting issues. In order to exploit most of these
vulnerabilities in Precurio's code base, a user account is required.
Precurio also includes a lot of third-party code though that is directly
accessible. These contain many vulnerabilities as well and do not
require any authentication. As always, an attacker needs only a single
exploitable issue for a successful attack. All third-party
vulnerabilities are inside of *Xinha* plugins.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post since there are no fixes available.

## Case Study

### Path Traversal to Code Execution

The most critical vulnerability is inside the plugin of a plugin.
*Xinha* is a WYSIWYG HTML editor and it is shipped together with
*ExtendedFileManager*. The file manager contains example code to
demonstrate how it is used. Unfortunately this example code is extremly
insecure.

```php
function makeFile($pathA, $pathB) {
    $pathA = Files::fixPath($pathA);
    if(substr($pathB,0,1)=='/')
        $pathB = substr($pathB,1);
    Return $pathA.$pathB;
```

```php
function processPaste() {
    switch ($_GET['paste']) {
        case 'copyFile':
            $src = Files::makeFile($this->getImagesDir(), $_GET['srcdir'].$_GET['file']);
            $dest = Files::makeFile($this->getImagesDir(), $_GET['dir']);
            return Files::copyFile($src,$dest,$_GET['file']);
        break;
        case 'moveFile':
            $src = Files::makeFile($this->getImagesDir(), $_GET['srcdir'].$_GET['file']);
            $dest = Files::makeFile($this->getImagesDir(), $_GET['dir'].$_GET['file']);
            return Files::rename($src,$dest);
        break;
```

The source and destination paths for copying and renaming a file in the
`ExtendedFileManager` are build from a static path location and
unsanitized user input. This code is vulnerable because the character
sequence `../` can be used to traverse the directory structure (*path
traversal*). As a result, attackers can point the destination of the
file operations to any directory on the system. We will demonstrate how
this can be abused shortly.

```php
function escape($filename) {
    Return preg_replace('/[^\w._]/', '_', $filename);
```

```php
function copyFile($source, $destination_dir, $destination_file, $unique=true) {
⋮
    $filename = Files::escape($destination_file);
    if (!copy($source, $destination_dir.$filename))
        return FILE_ERROR_COPY_FAILED;
⋮
function rename($oldPath,$newPath) {
⋮
    if (!rename($oldPath, $newPath))
        return FILE_ERROR_COPY_FAILED;
```

Note, that the regular expression in the `escape()` method prevents a
path traversal within the file name while it misses to sanitize the
directory. To avoid further security problems, the Xinha authors placed
a `.htaccess` file inside of the upload directory that removes the
handler for PHP files, i.e. they are not evaluated but directly
displayed. This way, a direct upload of PHP files is prevented.

```
<IfModule mod_php.c>
 php_flag engine off
</IfModule>
AddType text/html .html .htm .shtml .php .php3 .php4 .php5 .php6 .php7 .php8 .phtml .phtm .pl .py .cgi
RemoveHandler .php
RemoveHandler .php8
RemoveHandler .php7
RemoveHandler .php6
RemoveHandler .php5
RemoveHandler .php4
RemoveHandler .php3
```

There are multiple problems with this approach. First, not all
installations are using Apache as a web server and other servers could
ignore these instructions. Second, the blacklist might be missing a file
extension that is attached to the PHP interpreter in the web server
configuration. Third, we can use the `ExtendedFileManager` as described
above in order to rename the `.htaccess` file. Yes, it is that simple.
Once the file is disabled, an attacker can inject PHP code into the web
server log file and copy it to the upload folder with an PHP extension.
The vulnerability can also be used to copy configuration files into the
upload folder to get access to passwords and other private information.

## Time Line

| Date | What |
|------|------|
| 2016/09/20 | First try to contact vendor |
| 2016/10/21 | Second try to contact vendor |
| 2016/11/16 | Third try to contact vendor |

We tried to get in contact with the vendor both by e-mail and by web
form for almost 3 months with no response.

## Summary

A takeway from this analysis is to not use the Xinha plugin or
open-source software using it. It contains dangerous vulnerabilities and
is not maintained anymore. As a workaround, it is advised to remove the
Xinha plugin or, in case of Precurio, the directory
`public/library/xinha/plugins/ImageManager`, as already [considered by
the Xinha
authors](http://xinha.webfactional.com/ticket/1478).

> I think we should give consideration to just deleting these folders totally,
> over the last year I've had a number of instances of people coming to me
> with these folders filled with various malware.

Third-party code in general can introduce a big threat to your
application's safety. In order to increase security, libraries and
plugins should be stored outside of the web directory, i.e. it should
never be directly accessible by attackers. Using a dependency manager
like
[Composer](https://getcomposer.org/)
is also a good idea because it helps to easily update libraries and
warns about deprecated dependencies.
