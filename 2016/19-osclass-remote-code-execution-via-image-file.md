# osClass 3.6.1: Remote Code Execution via Image File

19 Dec 2016 by Robin Peraglie

In todays calendar gift, we present another beautiful chain of
vulnerabilities which, in the end, allows an attacker to remotely
execute arbitrary PHP code. This time, an attacker can smuggle his PHP
payload through a valid image file. The issues were detected by RIPS in
the open source marketplace software
[osClass](https://osclass.org/) 3.6.1 used for creating classifieds sites.

![osClass](images/osclass.png "osClass")

## RIPS Analysis

RIPS was able to scan the \~156,000 lines of code in just 23 seconds.
Looking at the scan results, a high number of vulnerabilities were
detected in this project. Especially *high*-rated vulnerabilities seem
to make the race. However, there is no *critical*-rated vulnerability
found on the spot.

The truncated analysis results are available in our RIPS demo
application. Please note that we limited the results to the issues
described in this post in order to ensure a fix is available.

## Case Study

In the following, we examine three vulnerabilities:

1.  Cross-Site Scripting
2.  File Write
3.  File Inclusion

By chaining these three vulnerabilities, the exploitation of the
cross-site scripting issue leads to remote code execution on a targeted
web server.

### Cross-Site Scripting

The cross-site scripting vulnerability can be triggered by an
authenticated administrator visiting a malicious link, as demonstrated
[in our previous
posts](/web/20201107032713/https://blog.ripstech.com/2016/freepbx-from-cross-site-scripting-to-remote-command-execution/).
Due to the generalized approach of input sanitization for HTML in
osClass's `getParam()` function, the parameter `country_code` is
insufficiently secured for a JavaScript context in line 409.

```php
<script type="text/javascript">
    show_region('<?php echo Params::getParam('country_code'); ?>',
    '<?php echo osc_esc_js(Params::getParam('country')); ?>');
```

Contrarily, in line 410, the parameter *country* is sanitized
sufficiently by using the `osc_esc_js()` function before printing. The
problem with the first approach is that an attacker can break out of the
quotes because they are not escaped by the `getParam()` function, as it
can be seen in the following code summaries.

```php
static function getParam($param, $htmlencode = false, $xss_check = true, $quotes_encode = true) {
    $value = self::_purify(self::$_request[$param], $xss_check);
    ⋮
static private function _purify($value, $xss_check) {
    ⋮
    self::$_config = HTMLPurifier_Config::createDefault();
    self::$_config->set('HTML.Allowed', '');
    ⋮
    $value = self::$_purifier->purify($value);
    ⋮
    return $value;
```

```php
function osc_esc_js($str) {
    ⋮
    $str = strip_tags($str, $sNewLines);
    $str = str_replace("\r", '', $str);
    $str = addslashes($str);
    $str = str_replace("\n", '\n', $str);
    $str = str_replace($aNewLines, '\n', $str);
    return $str;
```

Only `osc_esc_js()` escapes the single quotes in line 179 that can be
used to break out of the given context for the `country_code` parameter.

### File Write

Since osClass allows a user by default to upload images via AJAX, an
attacker can attach PHP code to the
[EXIF](https://en.wikipedia.org/wiki/Exif) data in form of an image description. It is important
to note that the image must be a valid image, as it will be rotated
internally by the application. An example for such a modified image
`muschel.jpg` can be observed in a hexeditor:

```
0000000: ffd8 ffe0 0010 4a46 4946 0001 0101 0060  ......JFIF.....`
0000010: 0060 0000 ffe1 00a8 4578 6966 0000 4949  .`......Exif..II
0000020: 2a00 0800 0000 0300 0e01 0200 6e00 0000  *...........n...
0000030: 3200 0000 2801 0300 0100 0000 0200 0000  2...(...........
0000040: 1302 0300 0100 0000 0100 0000 0000 0000  ................
0000050: 3c3f 7068 7020 6563 686f 2073 6865 6c6c  <?php echo shell
0000060: 5f65 7865 6328 2770 7764 3b6c 7320 2d6c  _exec('pwd;ls -l
0000070: 6127 293b 203f 3e48 494a 4b4c 4d4e 4f50  a'); ?>HIJKLMNOP
0000080: 5152 5354 5556 5758 595a 3241 4243 4445  QRSTUVWXYZ2ABCDE
0000090: 4647 4d4e 4f50 5152 5354 5556 5758 595a  FGMNOPQRSTUVWXYZ
00000a0: 3341 4243 4445 4647 4849 4a4b 4c4d 4e4f  3ABCDEFGHIJKLMNO
00000b0: 5051 5253 5455 5657 5859 5a31 3400 ffdb  PQRSTUVWXYZ14...
00000c0: 0043 0001 0101 0101 0101 0101 0101 0101  .C..............
00000d0: 0101 0101 0101 0101 0101 0101 0101 0101  ................
00000e0: 0101 0101 0101 0101 0101 0101 0101 0101  ................
00000f0: 0101 0101 0101 0101 0101 0101 0101 0101  ................
0000100: 0101 01ff db00 4301 0101 0101 0101 0101  ......C.........
0000110: 0101 0101 0101 0101 0101 0101 0101 0101  ................
0000120: 0101 0101 0101 0101 0101 0101 0101 0101  ................
0000130: 0101 0101 0101 0101 0101 0101 0101 0101  ................
0000140: 0101 0101 0101 0101 ffc0 0011 0800 0100  ................
0000150: 0103 0122 0002 1101 0311 01ff c400 1500  ..."............
0000160: 0101 0000 0000 0000 0000 0000 0000 0000  ................
0000170: 000a ffc4 0014 1001 0000 0000 0000 0000  ................
0000180: 0000 0000 0000 0000 ffc4 0014 0101 0000  ................
0000190: 0000 0000 0000 0000 0000 0000 0000 ffc4  ................
00001a0: 0014 1101 0000 0000 0000 0000 0000 0000  ................
00001b0: 0000 0000 ffda 000c 0301 0002 1103 1100  ................
00001c0: 3f00 bf80 01ff d9                        ?......
```

At address `0x050`, PHP code is placed into the EXIF data. This will
neither corrupt the image data nor its validaty, allowing the execution
of the code when `muschel.jpg` is included in PHP. By using the url
`index.php?page=ajax&action=ajax_upload`, an attacker can easily upload
certain files, such as images, to the server and the controller returns
the name of the newly uploaded file in the response body. Note that the
filename is not tainted and there is no possibility to upload PHP files
directly. In the following code lines, the upload is found in line 179
and the image rotation in line 180.

```php
case 'ajaxupload':
    ⋮
    $original = pathinfo($uploader->getOriginalName());
    $filename = uniqid("qqfile").".".$original['extension'];
    $result = $uploader->handleUpload(osc_content_path().'uploads/temp/'.$filename);
    $img = ImageResizer::fromFile(osc_content_path().'uploads/temp/'.$filename)->autoRotate();
    $img->saveToFile(osccontentpath().'uploads/temp/auto'.$filename, $original['extension']);
    $result['uploadName'] = 'auto'.$filename;
    echo htmlspecialchars(json_encode($result), ENT_NOQUOTES);
    break;
```

### File Inclusion

The administration module of osClass contains a local file inclusion
vulnerability. It is possible to include arbitrary files via the GET
parameter `plugin`. The following code lines are affected.

```php
switch ($this->action) {
⋮
	case 'error_plugin':
		⋮
		include( osc_plugins_path() . Params::getParam('plugin') );
		Plugins::install(Params::getParam('plugin'));
```

Not only that arbitrary files can be included when an administrator
visits a malicious link, but also this will install the inclusion
**persistently** in the database, as shown in the following code
summary.

```
static function install($path) {
    $data['s_value'] = osc_installed_plugins();
    $plugins_list    = unserialize($data['s_value']);
    ⋮
    $plugins_list[]  = $path;
    osc_set_preference('installed_plugins', serialize($plugins_list));
```

### Creating the Chain

By using the cross-site scripting vulnerability as an actuator, it is
possible to prepare a link with a JavaScript payload that in the end
automatically executes arbitrary PHP code on the targeted osClass web
server. When an authenticated administrator opens the prepared link, the
attached JavaScript code is reflected and executed in his browser, rides
the administrator session to upload a malicious image with ajax, and
then includes this image into PHP via the file inclusion vulnerability.

## Time Line

| Date | What |
|------|------|
| 2016/11/20 | First contact with vendor |
| 2016/11/21 | [Issues fixed in GitHub by vendor](https://github.com/osclass/Osclass/commit/ab46a18bc51953439dd4cdc8b9682fe28abd5fb2) |
| 2016/12/13 | [Vendor released fixed version](https://osclass.org/page/download) |

## Summary

RIPS presented a wide range of issues to the analyst of osClass in a
short period of time, allowing to choose an *escalation chain* from
these vulnerabilites quickly. Without automated analysis, the detection
and chain generation takes a large amount of time. We would like to
thank the osClass Team for quickly fixing the reported issues!
