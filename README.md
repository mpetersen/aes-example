Interoperable AES encryption with Java and JavaScript
=====================================================

In this post I will write about AES encryption and how to implement it interoperable between Java and JavaScript. For 
JavaScript I will use the [CryptoJS][1] library. I will not describe the AES algorithm in detail. Please use other 
sources for this.

Anyhow, let's start with the basics. The idea is, to encrypt a plaintext using a passphrase. However, for AES, more 
than that is required, you need:

*   a salt to generate the encryption key from the passphrase
*   the iteration count used during the salting process (here: fixed to 10000)
*   the padding mode (here: PKCS5) and a function to derive the key from a password (here: PBKDF2)
*   the initialization vector (IV)
*   the key length (here: 128bit)

Salt and IV are generated from a random number generator. This means, that both must be saved together with the 
ciphertext in order to be able to decrypt it later.

The encryption process requires 4 steps:

1.  Generate salt.
3.  Generate IV.
2.  Generate key using PBKDF2, passphrase, salt and the given key size and number of iterations (for the salting 
process).
4.  Encrypt the plain text using key and IV.

The decryption process is even simpler, because IV and salt have already been created (and have to be reused):

1.  Generate key (same as step 2. above).
2.  Decrypt cipher text using key and IV.

## JavaScript

Let's start with the JavaScript implementation:

        var salt = CryptoJS.lib.WordArray.random(16);
        var iv = CryptoJS.lib.WordArray.random(128/32);
        var key = CryptoJS.PBKDF2(password, salt, { keySize: 128/32, iterations: 10000 });
        var encrypted = CryptoJS.AES.encrypt(plaintext, key, { iv: iv });    

CryptoJS [returns an object][2], that contains the `key`, `iv`, `salt` and the `ciphertext`. The `ciphertext` is not a 
String, but a `WordArray` and you should convert it into text form before saving it, for example to Base64:

        var ciphertext64 = encrypted.ciphertext.toString(CryptoJS.enc.Base64);
    

To decrypt the ciphertext, again the key is created and then the decryption function called:

        var key = CryptoJS.PBKDF2(password, salt, { keySize: 128/32, iterations: 10000 });
        var cipherParams = CryptoJS.lib.CipherParams.create({
            ciphertext: CryptoJS.enc.Base64.parse(ciphertext64)
        });
        return CryptoJS.AES.decrypt(cipherParams, key, { iv: iv }).toString(CryptoJS.enc.Utf8);
    

## Java

The Java implementation looks quite different, but takes the same steps, except that the `Cipher` generates the IV and 
we need to get it, instead of passing a random IV to the algorithm:

        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
    
        // Generate salt
        Random random = new SecureRandom();
        byte[] salt = new byte[16];
        random.nextBytes(salt);
    
        // Generate key
        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
        KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 10000, 128);
        SecretKey key = new SecretKeySpec(factory.generateSecret(spec).getEncoded(), "AES");
    
        // Get IV from cipher
        cipher.init(Cipher.ENCRYPT_MODE, key);
        AlgorithmParameters params = cipher.getParameters();
        byte[] iv = params.getParameterSpec(IvParameterSpec.class).getIV();
    
        // Encrypt
        byte[] cipherText = cipher.doFinal(text.getBytes("UTF-8"));
    

Decryption is again a little bit simpler, because most parameters have already been created:

        // Generate key
        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
        KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 10000, 128);
        SecretKey key = new SecretKeySpec(factory.generateSecret(spec).getEncoded(), "AES");
    
        // Decrypt
        cipher.init(Cipher.DECRYPT_MODE, key, new IvParameterSpec(iv));
        String plainText = new String(cipher.doFinal(cipherText), "UTF-8");

 [1]: http://code.google.com/p/crypto-js
 [2]: http://code.google.com/p/crypto-js/#The_Cipher_Output