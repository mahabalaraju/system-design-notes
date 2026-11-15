Cryptography is the practise and study of techniques for secure communication in the presence of third parties. i.e.,  recieved and exclusively available for one.

during world war 2 it played crucial role -> enigma , alan turing colossus machine. 

today almost every social media( facebook, whatsapp, telegram ) bank credit card, blockchain based technologies ( bitcoin , ethereum ) 

ciphertext is the result of encryption performed on plaintext using an algorithm. 

ciphertext = encryptionFunction(plain_text, key) 

 plain_text = f-1(cipher_text ,key) 

1) symmetric cryptosystems:
  use just a single key both for encryption and for decryption as well --> its also called private key cryptosystem or symmetric cryptosystem. 
  
  private key must be exchanged.

 Ex: Caesar cipher , vigenere cipher, data encryption standard


2) Assymetric cryptosystem/ public key cryptosystems: 
   
   plain_text = f-1(cipher_text, private_key) -- decryption function 

   cipher_text = f(plain_text, public_key) -- encryption function 
   
   it takes lot of time to get private_key from the public_key and private keys never needs to be exchanged ( RSA or ECC ) 


cryptographic file formats:

## 1. What is a PEM File?
**PEM (Privacy-Enhanced Mail)** is the most common text-based format for storing and sharing cryptographic objects like public/private keys, SSL/TLS certificates, and certificate requests.

**Key Characteristics:**
* **Base64 Encoded:** Converts raw, unreadable binary cryptographic keys into readable text characters. This ensures the data can be safely copied, pasted, and transmitted via email or text documents without corruption.
* **Header and Footer:** Always includes dashed boundary lines to clearly identify the data type.

**Example Structure:**
```text
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA3T1Z... (Base64 encoded data) ...
-----END RSA PRIVATE KEY-----

## 4. Deep Dive: Why Use DER?
**DER (Distinguished Encoding Rules)** is the strict, raw binary foundation of most cryptographic files. If you open a DER file in a standard text editor, it will look like unprintable symbols or complete gibberish because it is fundamentally not meant for human readability.

### Why is it called "Distinguished"?
* **Mathematical Precision:** When creating a digital signature, the exact sequence of 1s and 0s matters. If even a single bit is out of place, the signature breaks and becomes invalid.
* **Zero Ambiguity:** DER provides a rigid set of rules guaranteeing that a specific piece of data can only be encoded in exactly *one* way. This leaves absolutely zero room for misinterpretation by different operating systems or programming languages.

### The PEM Connection
A PEM file is literally just a DER file that has been:
1. Translated into readable text characters (using Base64 encoding).
2. Wrapped in dashed header and footer lines (e.g., `-----BEGIN CERTIFICATE-----`) so you can easily copy and paste it.

### Where will you see it?
Because it is highly compact and mathematically strict, you will mostly encounter DER files (typically with `.der` or `.cer` extensions) working behind the scenes in:
* Java applications
* Windows environments
* Physical hardware (like smart cards)

