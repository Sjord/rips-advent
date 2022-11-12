# Day 13 - Epiphany

In this challenge we have a multi-part file upload and the uploaded file
has to belong to the content type `text/plain`, otherwise the file
upload is aborted (line 26). The developer tries to prevent that a user
uploads dangerous files like `.jsp` however the content type is
controlled by the attacker and thus this check can be easily bypassed.
Another user controlled input is the filename of the uploaded file
(`item.getName()` in line 28). It leads to a Path Traversal
vulnerability because a string like `/../` is a valid file name in Java.
Finally, we can upload any file we want because the content type check
is insufficient and via the Path Traversal vulnerability we can set the
destination of the uploaded file which leads to a Remote Command
Execution in the end.
