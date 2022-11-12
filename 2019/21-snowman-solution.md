# Day 21 - Snowman

In this code challenge, the method `decrypt` decrypts a user provided
hex-encoded cypher text using an insecure AES algorithm (line 27). The
previously encrypted cypher text is known to the attacker (line 10).
However, the attacker has no knowledge about the encryption key and
therefore the encrypted content is not known. Since the cypher text is
not protected by a MAC or a signature the attacker can manipulate the IV
(first 16 bytes of the cypher text) and abuse the CBC malleability to
cause a `BadPaddingException`.

The cypher text can be decrypted without knowing the key with the
Padding Oracle attack. In the worst case 16\*256 requests are needed to
decrypt one block. More information about the Padding Oracle attack can
be found
[here](https://www.owasp.org/images/e/eb/Fun_with_Padding_Oracles.pdf).
