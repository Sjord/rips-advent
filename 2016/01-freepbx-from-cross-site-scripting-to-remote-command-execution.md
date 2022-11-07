# FreePBX 13: From Cross-Site Scripting to Remote Command Execution

1 Dec 2016 by Hendrik Buchwald

[FreePBX](https://www.freepbx.org/)
is a web-based graphical user interface that helps
users to manage voice-over-IP services. With over one million production
systems using FreePBX worldwide it is the most widely deployed
open-source PBX (Private Branch Exchange) platform. Since FreePBX is
written completely in PHP, we decided to throw it into our code analysis
tool RIPS. The results were surprising...

## RIPS Analysis

The total amount of detected vulnerabilities is very high. Luckily, the
majority of the detected vulnerabilities are inside the administration
control panel, such that attackers either need to steal a valid account
first or they have to trick an administrator into visiting a malicious
website that triggers one of the critical vulnerabilities. For example,
a remote command execution vulnerability could be triggered by a less
critical cross-site scripting vulnerability. By chaining both
vulnerabilities, the severity is increased drastically and can lead to
full server compromise.

This is not necessary for all bugs though. In some cases, it is possible
to trigger vulnerable code without prior authentication. After we have
reported the vulnerabilities detected by RIPS, we were notified that a
file creation vulnerability (included in our reports) was abused in the
wild to infect and hack many FreePBX systems around the world. Hence, we
urge all FreePBX users to update their installation to the latest
version.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

A vulnerability that can be exploited directly may be more critical than
a vulnerability that requires user interaction - but it is also less
spectacular. For this reason, we present a combination of bugs that
demonstrate how vulnerabilities can be chained together to increase
their impact.

### Example 1: Remote Command Execution

One interesting way to inject arbitrary system commands in FreePBX lies
in the signature verification code. It was intended to make the system
more secure, but in the end it had the opposite effect. The following
code summaries highlight the *flow* of user input into a security
critical operation.

```php
$modulef->handleupload($_FILES['uploadmod']);
```

```php
public function handleupload($uploaded_file) {
    global $amp_conf;
⋮
    $filename = $amp_conf['AMPWEBROOT'] . '/admin/modules/_cache/' . $uploaded_file['name'];
    $this->_process_archive($filename);
```

```php
public function _process_archive($filename, $progress_callback = '') {
    FreePBX::GPG()->verifyFile($filename);
```

```php
public function verifyFile($filename, $retry = true) {
    $this->runGPG("–verify {$filename}");
```

```php
public function runGPG($params, $stdin = null) {
    $gpgdir = $this->getGpgLocation();
    $homediropt = "--homedir {$gpgdir}";
⋮
    $cmd = $this->gpg . " {$homediropt} " . $this->gpgopts . " --status-fd 3 {$params}";
    proc_open($cmd, $fds, $pipes, '/tmp', $this->gpgenv);
```

The superglobal variable `$_FILES` has two array keys that can be
tainted by a malicious user: the file name and its type. In this case,
the name of an uploaded file is concatenated unsanitized to an OS system
command in the `verifyFile()` method of the `GPG` class. As a result, an
attacker can inject arbitrary system commands into a file name which are
then executed with `proc_open()`.

Although the summarized data flow above looks straight-forward, it takes
effort to detect this vulnerability manually because the user input is
passed through thousands of lines of code in several different files and
functions until it is used in the security sensitive operation. RIPS was
able to identify the vulnerable data flow within seconds.

In order to exploit this vulnerability, an attacker needs to know the
circumstances that lead to the execution of the vulnerable code. In this
case there is a file extension check in the function
`_process_archive()`. The method `runGPG()` is only executed when the
file extension matches `gpg`. So one possibility to execute arbitrary
system commands is to upload a file that contains backticks in its name
and ends with `.gpg`, e.g. `` `command`.gpg `` .

The file name is also used to create a temporary cache file on the disk.
The file system (ext4) only supports names up to 255 characters, so if
the command is too long it will not be executed. Several other command
execution vulnerabilities were detected.

### Example 2: Cross-Site Scripting

FreePBX has a cross-site request forgery protection. If the action
parameter is set the referrer has to match the address of the server.

```php
if (!isset($no_auth) && $action != '' && $amp_conf['CHECKREFERER']) {
    if (isset($_SERVER['HTTP_REFERER'])) {
        $referer = parse_url($_SERVER['HTTP_REFERER']);
        $server = trim($_SERVER['SERVER_NAME']);
        $refererok = ($referer['host'] == $server);
    } else {
        $refererok = false;
    }
    if (!$refererok) {
        $display = 'badrefer';
    }
}
```

This prevents attackers from abusing existing sessions of logged-in
administrators because they are not able to spoof the referrer. While
this is not the best approach to prevent cross-site request forgery, it
is still effective. All remote command execution vulnerabilities that
were found have an action parameter and thus are protected by this piece
of code. However, RIPS also detected an abundance of cross-site
scripting vulnerabilities that do not have an action parameter and thus
can be used to nullify the referrer check.

Here is one simplified example of user input that is embedded into a
JavaScript context without proper sanitization.

```php
<?php $tech = htmlspecialchars($_REQUEST['tech']); ?>
<script>
var tech = '<?php echo !empty($tech) ? strtolower($tech) : strtolower($_REQUEST['tech']) ?>';
</script>
```

The request parameter `tech` is assigned to the `$tech` variable. Note,
that it gets encoded with `htmlspecialchars` which prevents the use of
the `&`, `"`, `<`, and `>` characters ^[1](#fn:3)^. Although in many
scenarios this would be sufficient to prevent cross-site scripting,
here, the user input is embedded into a JavaScript context with single
quotes. We can simply end the current string with a single quote and
inject arbitrary JavaScript instructions afterwards.

For exploitation, an attacker can embed and load an URL such as
http://target/admin/config.php?display=trunks&tech=rips%27;alert%281%29;%27%3C/script%3E
in the background of a web page. If they trick an administrator into
visiting the malicious page, the JavaScript code is printed by the PHP
code, executed in the context of the targeted user, and thus the
attacker can perform any action in the FreePBX administration panel
effectively bypassing the referrer check. The payload is converted to
lower case characters though, which makes it a little more difficult to
exploit because we cannot call JavaScript functions that contain upper
case letters (i.e. most of them) - but it is certainly possible as
demonstrated in the [following video](images/01-freepbx.webm).

The final payload does not require upper case nor encoded letters and
the file name has a length of 163 characters. All filters are bypassed
and the command execution vulnerability is triggered by a click of our
proof-of-concept link. The full payload will not be published.

## Summary

There are many vulnerabilities in FreePBX that can have diverse reasons.
For one, the project is in development for more than 12 years. Everyone
that worked on a project for a longer time knows that priorities shift,
unforeseen requirements emerge and the landscapes of languages change.
Old libraries are replaced by new libraries, different coding styles and
patterns come into fashion and the like.

With close to 600,000 lines of code, FreePBX is also a large and complex
project. Too large and complex for a single person to know all details
and specifics. There are many different people writing and maintaining
modules and core systems of FreePBX. This makes it difficult to review
all changes in full detail and bad code slips through from time to time.
Unlike many other types of bugs, security vulnerabilities often remain
unnoticed unless they are triggered on purpose. Furthermore, there are
many different types of vulnerabilities that require in-depth studying
of the attackers possibilities and effective remmediation methods.

As writing secure software is challenging and mistakes will always
happen, an important thing is how vendors react when notified about
problems. We would like to thank the team of FreePBX at Sangoma
Technologies Corporation [that worked with
us](https://www.freepbx.org/building-a-more-secure-communications-platform/)
on resolving the issues. They responded fast, professional, and fixed
the most critical vulnerabilities in a fast manner.
