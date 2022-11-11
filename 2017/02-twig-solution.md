# Day 2 - Twig

The challenge contains a cross-site scripting vulnerability in line 26.
There are two filters that try to assure that the link that is passed to
the `<a>` tag is a genuine URL. First, the `filter_var()` function in
line 22 checks if it is a valid URL. Then, Twig's template escaping is
used in line 10 that avoids breaking out of the `href` attribute.

The vulnerability can still be exploited with the following URL:
`?nextSlide=javascript://comment%250aalert(1)`.  
The payload does not involve any markup characters that would be
affected by Twig's escaping. At the same time, it is a valid URL for
`filter_var()`. We used a JavaScript protocol handler, followed by a
JavaScript comment introduced with `//` and then the actual JS payload
follows on a newline. When the link is clicked, the JavaScript payload
is executed in the browser of the victim.
