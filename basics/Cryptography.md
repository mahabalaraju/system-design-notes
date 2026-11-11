Cryptography is the practise and study of techniques for secure communication in the presence of third parties. i.e.,  recieved and exclusively available for one.

during world war 2 it played crucial role -> enigma , alan turing colossus machine. 

today almost every social media( facebook, whatsapp, telegram ) bank credit card, blockchain based technologies ( bitcoin , ethereum ) 

ciphertext is the result of encryption performed on plaintext using an algorithm. 

ciphertext = encryptionFunction(plain_text, key)  plain_text = f-1(cipher_text ,key) 

1) symmetric cryptosystems:
  use just a single key both for encryption and for decryption as well --> its also called private key cryptosystem or symmetric cryptosystem. 
  
  private key must be exchanged.

 Ex: Caesar cipher , vigenere cipher, data encryption standard


2) Assymetric cryptosystem/ public key cryptosystems: 
   
   plain_text = f-1(cipher_text, private_key) -- decryption function 
   cipher_text = f(plain_text, public_key) -- encryption function 
   
   it takes lot of time to get private_key from the public_key and private keys never needs to be exchanged ( RSA or ECC ) 
 
