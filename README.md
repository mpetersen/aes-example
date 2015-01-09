Interoperable AES encryption with Java and JavaScript
=====================================================

AES implementations are available in many languages, including Java and JavaScript. In Java, the `javax.crypto.*` 
packages are part of the standard, and in JavaScript, the excellent [CryptoJS][1] provides an implementation for many 
cryptographic algorithms. However, due to different default settings and various implementation details, it is not 
trivial to use the APIs in a way, that the result is the same on all platforms.

This example demonstrates implementations of the algorithm in Java and JavaScript that produces identical results using
passphrase based encryption. For AES encryption, you cannot - or shouldn't - simply use a password in order to encrypt
data. Instead, many parameters need to be defined, such as:

* iteration count used for the salting process
* padding mode
* key derivation function
* key length

Then, additional initialization parameters need to be defined, such as the salt and the initialization vector (IV). With
all parameters defined, the encryption process is the same for both, Java and JavaScript:

1. Generate salt and IV (this is typically done using a secure psuedo-random number generator; in my example tests both
are fixed in order to produce predictable results).
2. Generate the key (using the PBKDF2 function) from the given passphrase, salt, key size and number of iterations (for
the salting process.
3. Encrypt the plaintext using key and IV.

The decryption process is even simpler, because IV and salt have already been generated. These have to be reused to 
successfully reproduce the plaintext. Therefore, for successful encryption, you have to store IV, salt and
iteration count (as long as it is not fixed for your application) along with the cipher text. Since these parameters 
don't need to get generated the decryption process only has 2 steps:

1.  Generate key (same as step 2. above).
2.  Decrypt cipher text using key and IV.

In this example, I have created a utility class for each language: `AesUtil.java` and `AesUtil.js`. In the test, all
data (salt, passpharse, IV, plaintext, ciphertext) are represented as String. The ciphertext is encoded using base64,
in order to get a proper and compact representation of the bytes (AES produces a byte array, not a String). The other
parameters, salt and IV are encoded in hex. This is useful to effectively count and read the number of bytes used
(and see if the length of both parameters is correct).

## JavaScript implementation `AesUtil.js`

1. Generate key:

          var key = CryptoJS.PBKDF2(
              passPhrase, 
              CryptoJS.enc.Hex.parse(salt),
              { keySize: this.keySize, iterations: this.iterationCount });

    > Note, that `this.keySize` is the size of the key in 4-byte blocks. So, if you want to use a 128-bit key, you have to 
    divide the number of bits by 32 to get the key size used for CryptoJS.

2. Encrypt plaintext:

    The object returned by the `encrypt` method is not a String, but a object that contains the parameters of the algorithm 
    and the ciphertext.
    
          var encrypted = CryptoJS.AES.encrypt(
              plainText,
              key,
              { iv: CryptoJS.enc.Hex.parse(iv) });

    To convert the encryption result into base64 format, you have to use the `toString()` function:
    
          var ciphertext = encrypted.ciphertext.toString(CryptoJS.enc.Base64);

3. Decrypt ciphertext:

    To decrypt, a parameter object is created first, that contains the ciphertext (note base64 encoding is used here):
    
          var cipherParams = CryptoJS.lib.CipherParams.create({
            ciphertext: CryptoJS.enc.Base64.parse(ciphertext)
          });
          var decrypted = CryptoJS.AES.decrypt(
              cipherParams,
              key,
              { iv: CryptoJS.enc.Hex.parse(iv) });
    
    Again, to get the result in text form, you use the `toString()` function:
    
          var plaintext = decrypted.toString(CryptoJS.enc.Utf8);

## Java implementation `AesUtil.java`

The Java implementation looks a bit different, but the structure is the same:

1. Create a `cipher` instance:

            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");

2. Generate key:

            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
            KeySpec spec = new PBEKeySpec(passphrase.toCharArray(), hex(salt), iterationCount, keySize);
            SecretKey key = new SecretKeySpec(factory.generateSecret(spec).getEncoded(), "AES");

3. Encrypt:

            cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec(hex(iv)));
            byte[] encrypted = cipher.doFinal(bytes);

4. Decrypt:

            cipher.init(Cipher.DECRYPT_MODE, key, new IvParameterSpec(hex(iv)));
            byte[] decrypted = cipher.doFinal(bytes);


## Running the tests

The project uses Maven as build environment. After cloning the repository, you just need to type:

    #> mvn test
    
That executes both, the Java unit tests and the JavaScript Jasmine specs.

## Browser example

To run a simple example to encrypt a text in the browser and send it to a servlet, you just need to run:

    #> mvn jetty:run
    
Then open [http://localhost:8080][2]. The example encrypts a text using a password, which is then sent to
the server. The request contains everything required to encrypt the password, such as salt, IV, iteration count,
and the passphrase. In the real world, you need to pass all these to the server, except passphrase, of 
course. 

 [1]: http://code.google.com/p/crypto-js
 [2]: http://localhost:8080