# Day 14 - Chimney

This servlet exports a CSV file which contains unfiltered user input,
and thus it contains a Formular Injection vulnerability. The parameter
`description` in line 22 flows into the variable `rows` and finally into
the `StringBuilder` in line 33. However, we have no sanitization and an
attacker can inject a payload like `=cmd|'/C calc.exe'!Z0` via
`description` that finally leads to a exported CSV file of the
structure:

```
Name,Role,Description
Scott,editor,=cmd|'/C calc.exe'!Z0
```

When a spreadsheet program such as Microsoft Excel or LibreOffice Calc
is used to open the file, any cells starting with `=` will be
interpreted by the software as a formula. An attacker could exfiltrate
data / execute OS commands, or do other malicious actions.

[Source](https://www.owasp.org/index.php/CSV_Injection)
