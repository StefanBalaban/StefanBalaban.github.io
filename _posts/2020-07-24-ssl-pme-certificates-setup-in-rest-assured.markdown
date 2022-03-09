---
layout: post
title:  "SSL PME certificates setup in REST Assured"
date:   2020-07-24 10:05:34 +0100
categories: java ssl
---
Hi everyone,
so recently I was working on a test automation framework using REST-assured that calls an endpoint that requires the use of an SSL certificate.
The SSL certificate was in the PEM format (.crt and .key file) and the code was in Java. 
This meant importing the PEM files into a JKS keystore so it could be used in the Java project and creating a helper method to be used during the HTTP request setup.

This guide is for Windows.
# Keystore

First you need Java7+ and OpenSSL, after downloading and installing them, add both to your PATH Environment Variables.

Obtain the PEM certificate (.pem, .cer, crt) and its private key (.key). Place the files in a single directory
You'll also need the private keys passphrase, lets say in this case the passphrase is abcd.

For the keystore password, I'll be using 123456, it can be anything else but it needs to be at least 6 characters long.

Open the command-line — I'm using PowerShell but you can use any other; and navigate to the folder where you are storing the files, for example, let's say its under the C:\User\user.name\cert directory.

Optionally, at first, you can add the certificate to your keystore by running the following command:

```powershell
PS C:\Users\user.name\cert> keytool -import -file certificate.crt -alias example -keystore javaKeyStore.jks
```

After running the command you will be prompted for the password, enter `123456`.
The command-line will display the certificate information and ask you do you want to add it to your keystore:

```powershell
Trust this certificate? [no]:  yes
```
Moving on now lets create the keystore.
First create a p12 keystore and after which you can create a JKS keystore.

Run the following command:

```powershell
openssl pkcs12 -export -in certificate.crt -inkey privateKey.key -certfile certificate.crt -name "certname" -out keystore.p12
```

You will be prompted for the passphrase so enter `abcd` after which you will be prompted for the password so enter `123456`

If you look in your directory you'll see the keystore.p12 file you just created.

Next, enter the following command to create the JKS keystore:

```powershell
keytool -importkeystore -srckeystore keystore.p12 -srcstoretype pkcs12 -destkeystore keystore.jks -deststoretype JKS
```

You will be prompted for the password so enter `123456` . 
After doing so you will see the keystore.jks file in the directory.

# Project

For this example I have placed the keystore inside the project in the resources folder.

```java
import java.io.FileInputStream;
import java.security.KeyManagementException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.UnrecoverableKeyException;

import io.restassured.config.SSLConfig;

public class CertHelper {

  public static SSLConfig getSslConfig() {
    String password = "123456";
    KeyStore keyStore = null;
    try {
      FileInputStream fis = new FileInputStream("src/main/java/resources/keystore.jks");
      keyStore = KeyStore.getInstance("JKS");
      keyStore.load(
          fis,
          password.toCharArray());

    } catch (Exception ex) {
      System.out.println("Error while loading keystore >>>>>>>>>");
      ex.printStackTrace();
    }
    if (keyStore != null) {
      org.apache.http.conn.ssl.SSLSocketFactory clientAuthFactory = null;
      try {
        clientAuthFactory = new org.apache.http.conn.ssl.SSLSocketFactory(keyStore, password);
      } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
      } catch (KeyManagementException e) {
        e.printStackTrace();
      } catch (KeyStoreException e) {
        e.printStackTrace();
      } catch (UnrecoverableKeyException e) {
        e.printStackTrace();
      }
      return new SSLConfig().with().sslSocketFactory(clientAuthFactory).and().allowAllHostnames();
    }
    else return  null;
  }
}
```
Finally to use the certificate you have to assign it to the config property of RestAssured. 
```java
RestAssured.config = RestAssured.config().sslConfig(CertHelper.getSslConfig());
```
And that is all.
# Notes
If you have some trouble regarding the certificate you might need to change the passphrase of the private key (`abcd`) to be the same as the keystore password (`123456`).

