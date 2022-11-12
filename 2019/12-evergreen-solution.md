# Day 12 - Evergreen

A class is assigned to the div-element in line 5 dynamically,
originating from the variable `customClass`. Said variable is
initialized with the static string `default` but is then overwritten in
init.jsp when it is included in line 3. Since init.jsp fetches the user
input and assigns it directly to `customClass` without any sanitization
or verification, the contents of the div's class attribute can be
controlled by an attacker. Breaking out of the attribute is trivial by
using a double quote. All other instances of user input are propertly
sanitized using an ESAPI encoder.
