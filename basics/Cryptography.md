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
