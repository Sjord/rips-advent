# Introducing the RIPS analysis engine

4 Dec 2016 by Johannes Dahse

In today's post, we would like to share some insights into our static code analysis
engine RIPS that detected the security bugs described in the previous
and upcoming calendar gifts. The engine has a long history and went
through several generations before reaching its current performance.
What does it actually do within the few seconds after you click on the
*scan* button and the first vulnerability report pops up? How can a
security vulnerability be automatically detected in source code? Let's
have a look.

## History

### 2007 - 2009

Almost 10 years ago, a simple PHP Scanner was developed during
popularity gaining [Capture The
Flag](https://ctftime.org/ctf-wtf/) (CTF) hacking battles of university teams. The scanner
based on regular expressions and identified simple connections between
user input that is first assigned to a variable and then used in a
critical operation of the PHP code. It worked for the analysis of small
applications in CTF events, but it became quickly evident that regular
expressions are insufficient for parsing a programming language
thoroughly.

```php
// this examples confuses even the syntax highlighter of our blog
$var = rips1("rips2(/* rips3())", rips4("*/"));
// the function rips1() and rips4() is called, rips2() within a string is not 
```

### 2009 - 2012

Using a tokenizer was the first step into the right direction. A new
tool was developed that first splits the PHP code into its single tokens
following the official PHP syntax, leading then to much more precise
analysis results. The tool was named RIPS and released during the [Month
of PHP
Security](http://php-security.org/2010/05/24/mops-submission-09-rips-a-static-source-code-analyser-for-vulnerabilities-in-php-scripts/) (MOPS). Today, it is the [most
popular](https://sourceforge.net/projects/rips-scanner/files/stats/timeline?dates=2010-05-23+to+2016-12-04) open-source PHP analysis tool used by many leading
companies world-wide for security audits. The major drawback of the
open-source version is, however, the vast amount of false positives and
the missing support for analyzing object-oriented code that is used in
every modern PHP application.

```php
class Text {
    public function __construct($data) {
        $this->data = $data;
    }
}
$t = new Text($_POST['data']);
echo $t->data; // XSS
```

### 2012 - 2016

To overcome these limitations, a new analysis engine was built from
scratch and that leverages the lessons learned during the past years of
engineering. Challenges of the dynamic PHP language and its features
were tackled and the efficient analysis of large web applications with
object-oriented PHP code was pioneered by refining state-of-the-art
static code analysis techniques with novel approaches specifically
designed for the PHP language. As of today, RIPS is the only SAST tool
with a dedicated focus on PHP analysis from its start and, as a result,
is able to detect even complex vulnerability types with high precision.

## How it works

When the new engine is pointed to a code repository, it transforms all
PHP code into a graph representation within an initial analysis phase.
For this purpose, the code is split into its single tokens, abstract
syntax trees are built and devided into blocks, and then these blocks
are connected to an annotated control flow graph. Now the data *flow*
can be analyzed on top of this abstract model. With the help of taint
analysis, user input is detected that is used unsanitized in a security
critical operation of an application by following the data flow of each
input throughout the graph model. The concept of a *source* tainting a
*sink* can be applied to many different vulnerability types, such as
cross-site scripting (XSS) and SQL injection.

### Example

In the following, we have a look at a simple example code and its
analysis. We skip all obstacles that stem from inter-procedural
(functions, methods), constraint, or object-sensitive analysis. The
example will demonstrate why a dedicated focus on PHP and its features
is necessary in order to detect and validate a security vulnerability.

```php
$id = $_POST['id'];
if(...) {
    $id = (int)$id;
}
else {
    $id = htmlentities($id);
}
echo "<div id='$id'>"; // XSS
```

The code contains an XSS vulnerability in line 8 because the user input
/ source (`$_POST['id']`) in line 1 *flows* into the sensitive sink
`echo`. In between, input sanitization is applied which requires further
analysis. In the initial analysis phase, the code is parsed by the
engine and tokenized. It identifies different branches (`if`/`else`) and
separates the code into different blocks accordingly. These block are
then connected to a control flow graph with labeled edges.

Each block of the graph is analyzed for sensitive sinks. In our example,
the `echo` operator is detected in the last block (red border). At this
point, the engine invokes a markup parser dedicated to the markup of the
sink - for our example, HTML. Our HTML markup parser is able to pinpoint
the exact location of dynamic content within the HTML. It detects that
the variable `$id` lies within a **single-quoted** attribute of an HTML
element. This information is very important because now we acknowledge
what an attacker needs in order to break out of the attribute and to
inject malicious HTML: he needs a single-quote `'`.

Next, the engine resolves the arguments of the sink `echo` from the
previous blocks. The variable `$id` is looked up in the left block where
a typecast prevents any exploitation and stops the trace. Then, the
variable is looked up in the right block. Here, the PHP built-in
function `htmlentities()` is used to sanitize `$id`. The engine executes
a complete simulation of this built-in function and detects that without
an [additional
parameter](http://php.net/htmlentities),
only `"`, `<`, and `>` characters are encoded to HTML entities. Without
this PHP-specific precision, previous generations as well as other
approaches would stop the trace at this point and often whitelist
`htmlentities()` as an XSS sanitizer. Instead, our engine learns during
simulation precisely which characters are affected, and continues the
trace from the right block to the first block in our graph. Here, the
variable `$id` maps to a `$_POST` source.

Finally, our engine can combine all gathered information and decide that
the source `$_POST['id']` is not sanitized against single-quotes and
taints an HTML attribute `id` with single-quotes as delimiter. Because
of insufficient sanitization, an attacker can perform cross-site
scripting attacks and a vulnerability report
is issued with the following facts. Further, the severity can be
fine-tuned based on the vulnerability type, its markup context, the type
of source, and any present security mechanisms.

#### Cross-Site Scripting (single-quoted attribute)

Severity: Medium, CWE:
[79](https://cwe.mitre.org/data/definitions/79.html),
OWASP Top 10:
[A3](https://www.owasp.org/index.php/Top_10_2013-Top_10),
SANS 25 Rank:
[4](https://www.sans.org/top25-software-errors/)

```php
$id = $_POST['id'];
⋮
$id = htmlentities($id);
⋮
echo "<div id='$id'>";
```

**Reconstructed HTML Context**

```html
<div id='$_POST['id']'>
```

## Summary

The PHP landscape changed in the past years and so did the requirements
for SAST tools. Diverse language features and characteristics, as well
as more security-aware developers and growing code sizes lead to more
complex applications. Static code analysis has to be advanced in order
to keep up with these challenges for the automated detection of security
issues.

In this post, we had a glance at the inner working of the RIPS analysis
engine and at some key advances over previous generations. We hope that
we provided some insights into the world of code analysis that will be
helpful for understanding the background of our upcoming vulnerability
posts. In case you would like to work together with leading experts in
the field of static analysis, we are currently
hiring and are looking forward to getting in
contact with you.
