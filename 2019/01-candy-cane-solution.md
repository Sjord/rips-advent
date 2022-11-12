# Day 1 - Candy Cane

The function `extractString` opens an OpenOffice document which was
previously uploaded by an attacker and extracts the text from it. An
OpenOffice document is actually a ZIP file which contains multiple
resources. Amongst other things the document contains a file named
content.xml which contains the text information in XML representation.
This file is parsed by the method `org.jdom2.input.SAXBuilder.build()`
which is vulnerable to XXE injection. We can exploit this XXE by adding
the following DOCTYPE declaration to the XML document and reference the
XXE entity in the document. This will insert the contents of the file
/etc/passwd into the document.
