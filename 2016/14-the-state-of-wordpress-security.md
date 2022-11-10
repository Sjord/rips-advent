# The State of Wordpress Security

14 Dec 2016 by Hendrik Buchwald

Plugins from the community are an integral part of most Wordpress sites.
We downloaded all **47,959** plugins that are available from the
*official Wordpress repository* and analyzed them with our static code
analyzer RIPS. Shockingly, about every second larger plugin contains at
least one medium severity issue. But is it really that bad?

## Statistics

Before we start analyzing the vulnerabilities, let us have a look at the
general statistics to understand what the results really indicate. Our
scan includes all plugins that are hosted in [the official Wordpress
repository](http://plugins.svn.wordpress.org/) and have at least one PHP file. If there are
releases, we use the latest release, otherwise we use the code from [the
trunk](https://en.wikipedia.org/wiki/Branching_(version_control)#Development_branch).
There are 44,705 plugins that fulfill this criteria.
The average amount of files per plugin is 8.43 and the average amount of
lines per plugin is 602. As concluded from the following statistics, the
majority of the plugins are very small. For example, over 14,000 plugins
(32%) consist of only 2-5 files.

### Issue Distribution

As stated in our introduction, there are 10,523 *larger* plugins with
more than 500 lines of code and 4,559 of them (43%) contain at least one
medium severity issue, such as cross-site scripting. But how are the
issues spread among the plugins? To answer this question, we calculated
the number of plugins with no issues, only low severity issues, medium
severity and below issues, high severity and below issues, and critical
severity and below issues.


The result indicates that a vast majority of plugins do not have any
vulnerabilities at all. Given a total amount of 67,486 detected security
issues this means that the plugins that do have at least one security
issue must have a lot of them. Our hypothesis is that this is the case
because most plugins are very small. It is much harder to introduce a
vulnerability in 100 lines of code than it is in 5,000 lines of code. To
verify our hypothesis we calculated the relation of lines of code to
number of issues.


The blue dots show that most plugins have less than 1,000 lines of code.
The orange dots on the other hand show that the plugins with less than
1,000 lines of code have close to zero issues on average. When the blue
dots start approaching 1, the orange dots start to spread out. This
supports our theory that most plugins do not have vulnerabilities
because they are small.

### Issue Type Distribution

Another important statistic is the distribution of issue types in order
to understand the severity of the detected issues in the plugins.


\

Unsurprisingly, the most common issues are cross-site scripting
vulnerabilities which occur whenever user input is printed without
proper sanitization to the HTML response page. For one, these issues
appear frequently because the output of data is the [most common
operation of PHP applications](https://homepages.cwi.nl/~jurgenv/papers/ISSTA-2013.pdf) and thus more affected by
security violations than other operations. And second, given the
[diversity of HTML contexts and its
pitfalls](/web/20201108005609/https://blog.ripstech.com/2016/introducing-the-rips-analysis-engine/)
in sanitization these issues are easily added. Cross-site scripting
vulnerabilities are quite serious in Wordpress because they can be used,
[for
example](/web/20201108005609/https://blog.ripstech.com/2016/freepbx-from-cross-site-scripting-to-remote-command-execution/),
to inject PHP code through the template editor. Luckily, they do require
interaction with an administrator though.

The second most common issues are SQL injection vulnerabilities. They
are even more severe than cross-site scripting vulnerabilities because -
in the worst case - they can be used to extract sensitive information
from the database - [for example
passwords](/web/20201108005609/https://blog.ripstech.com/2016/teampass-unauthenticated-sql-injection/) -
without any user interaction at all. As a result they can be used for
fully automated attacks.

All other issue types are comparatively rare and negligible in the
overall picture.

### Popular Targets

Another information we were interested in is what plugins attackers are
trying to exploit most actively at the moment. We are running a small
Wordpress honeypot for quite some time know and could extract the
information from our logs. Overall, over 200 attacks were recorded from
January of 2016 to December of 2016, related to the following Wordpress
plugins:

- Revolution Slider: 69 attacks
- Beauty & Clean Theme: 46 attacks
- MiwoFTP: 41 attacks
- Simple Backup: 33 attacks
- Gravity Forms: 11 attacks
- Wordpress  Marketplace: 9 attacks
- CP Image Store: 8 attacks
- Wordpress Download Manager: 6 attacks

All attacks relate to publicly known vulnerabilities that [are well
documented](https://blog.sucuri.net/2014/09/slider-revolution-plugin-critical-vulnerability-being-exploited.html).
Most of the vulnerabilities are very easy to
exploit and allow the execution of arbitrary code which makes them
interesting for the creation of PHP-based botnets. Additionally, we
observed frequent brute-force attempts on the administration area of the
Wordpress honeypot blog.

## Case Study

As we saw in the previous section, Wordpress plugins suffer from all
kinds of typical web vulnerabilities. In this section, we will describe
some of our uncovered issues in detail. We reported all detected
vulnerabilities earlier this year and patches are available.

### All In One WP Security & Firewall

The first plugin that will be analyzed in detail is called *All In One
WP Security & Firewall*. It adds some additional layers of security to
Wordpress, for example a brute force protection for the login or file
permission checks. There are definitely quite a lot of good ideas
integrated into this plugin, but some functionality cuts both ways.
Meaning, it closes some attack vectors and opens new ones. Take the file
permission manager for example. It is intended to change file
permissions to something secure, but if an attacker gets access to the
administration panel it can also be used to change the file permissions
to something insecure, i.e. make read-only files writable. Another
dangerous function is the ability to backup and restore the Wordpress
configuration files because it can be used to inject PHP code into
`wp-config.php`. In combination with the file permission manager, a code
execution is guaranteed in case there is a cross-site scripting
vulnerability somewhere on the page. Indeed, RIPS detected one in the
plugin itself. In the end, the usage of this security plugin can bring
more security risks than running Wordpress without it.

```php
var $menu_tabs_handler = array(
    'tab1' => 'render_tab1',
    'tab2' => 'render_tab2',
    'tab3' => 'render_tab3',
    'tab4' => 'render_tab4',
    'tab5' => 'render_tab5',
);
⋮
function render_menu_page() {
    $tab = $this->get_current_tab();
    $this->render_menu_tabs();
    call_user_func(array(&$this, $this->menu_tabs_handler[$tab]));
```

```php
function get_current_tab() {
    $tab_keys = array_keys($this->menu_tabs);
    $tab = isset($_GET['tab']) ? sanitize_text_field($_GET['tab']) : $tab_keys[0];
    return $tab;
```

```php
public function render_tab3() {
⋮
    echo '<input type="hidden" name="tab" value="' . $_REQUEST['tab'] . '" />';
```

This cross-site scripting vulnerability is quite interesting because at
the first look this does not seem to be exploitable. After all, the
function `render_tab3()` is only called when the parameter `tab` is
*tab3*. We can still exploit this because the function
`get_current_tab()` uses `$_GET['tab']` whereas `render_tab3()` uses
`$_REQUEST['tab']`. The super global `$_REQUEST` is a combination of
`$_GET`, `$_POST` and `$_COOKIE`. So what happens if we send `tab` as
both a GET and a POST variable? The POST value overwrites the GET value,
but only inside of `$_REQUEST`. So all we have to do is to send a
cross-site scripting payload as POST parameter `tab` to
`admin.php?page=aiowpsec&tab=tab3`.

The cross-site scripting vulnerability is of course fixed by now, and so
are other issues that we reported. Some of the issues still exist
though, because the only way to fix them is by removing the
functionality completely.

### Podlove Publisher

The second plugin that will be dissected is called *Podlove Publisher*,
a Wordpress plugin to manage podcasts. It suffered from multiple SQL
injections and cross-site scripting vulnerabilities (funnily enough also
in a parameter named `tab`) that are fixed by now. The SQL injections
were all caused by the following code.

```php
private function save() {
    $feed = \Podlove\Model\Feed::find_by_id( $_REQUEST['feed'] );
    $feed->update_attributes( $_POST['podlove_feed'] );
```

```php
public function update_attributes( $attributes ) {
⋮
    foreach ( $attributes as $key => $value )
        $this->{$key} = $value;
⋮
    return $this->save();
```

```php
public function save() {
    global $wpdb;
    if ( $this->is_new() ) {
        $this->set_defaults();
        $sql = 'INSERT INTO '
             . static::table_name()
             . ' ( '
             . implode( ',', self::property_names() )
             . ' ) '
             . 'VALUES'
             . ' ( '
             . implode( ',', array_map( array( $this, 'property_name_to_sql_value' ), self::property_names() ) )
             . ' );'
        ;
        $success = $wpdb->query( $sql );
        if ( $success ) {
            $this->id = $wpdb->insert_id;
        }
    } else {
        $sql = 'UPDATE ' . static::table_name()
             . ' SET '
             . implode( ',', array_map( array( $this, 'property_name_to_sql_update_statement' ), self::property_names() ) )
             . ' WHERE id = ' . $this->id
        ;
        $success = $wpdb->query( $sql );
    }
```

The author tried to save some work by dynamically setting properties of
the model from user input, called *mass-assignment*. The idea is not bad
in general, but care should be taken when every property of an object
can be tainted with user input, even properties that are not supposed to
be set by the user, like the `id`. Usually, the ID is supposed to be an
integer value that stems from `insert_id`, but the mass-assignment
allows an attacker to overwrite and use it to extend the SQL query and
retrieve sensitive information from the database. All other properties
are escaped by `property_name_to_sql_update_statement()`.

On a positive note, all vulnerabilities in this plugin were fixed very
fast and a secure version was available after only 2 days. This was one
of the fastest responses we experienced so far, so if you are searching
for a Wordpress podcast plugin, give it a try.

## RIPS ❤ Wordpress

We incorporated detailed knowledge about Wordpress internals into the
RIPS engine. There are multiple reasons why this is important. For
example, consider the following hook.

```php
public function custom_search_where($where) {
    $term = get_query_var('s');
    if (!empty($term)) {
        $where .= ' AND category = ' . $term;
    }
    return $where;
}
add_filter('posts_where', 'custom_search_where', 1, 1);
```

The hook modifies the `WHERE` conditions of database queries and makes
them vulnerable by appending user input from the Wordpress function
`get_query_var()`. For a human reader this is pretty obvious - for a
computer it is not. The user input never touches a sensitive function
like `mysql_query()` directly, so only if the analyzer knows that the
return values of `posts_where` hooks are used inside of a sensitive
function it can detect the vulnerability.

The core of Wordpress has 2,472 locations that can be
[hooked](https://developer.wordpress.org/reference/hooks/). Furthermore, Wordpress provides many functions to
encode or escape variables, such as `esc_sql()`, that are
commonly used in plugins. Due to a strong dedication to PHP and
intensive research, RIPS supports the security analysis of complex
platforms such as Wordpress with all its functionalities and oddities.
Hence, it is possible to analyze a Wordpress plugin without analysis of
the Wordpress core itself. This allowed us to scan 44,705 plugins on a
single machine within one day.

## Summary

Wordpress is not as insecure as its reputation would suggest. Rather it
is a top target due to its incredible prevalence. Yes, there are a lot
of vulnerabilities in the Wordpress ecosystem, but most of them are in a
small percentage of the plugins. While many plugins do not contain
vulnerabilities at all because of their small size, the ones that do
have issues, have a lot of them. The more lines of code a plugin has,
the more vulnerabilities it has on average. Please note that we do not
claim to have found all vulnerabilities that possibly exist in all
plugins nor that we can guarantee the
[exploitability](/web/20201108005609/https://blog.ripstech.com/2016/non-exploitable-security-issues/)
of the detected issues. As a take-away of this post, we recommend to
install only plugins that you really need, to keep all plugins up to
date, and to choose strong passwords.
