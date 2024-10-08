## About
The guide explains how to create a Java KeyStore (JKS), create a self signed certificate and then place it inside of the Java KeyStore.

## What is a Java KeyStore
A Java KeyStore is a secure, digital vault for storing cryptographic information like private keys, certificates, and public keys. It's like a password-protected container that keeps your sensitive data safe.   

The JDK includes a tool called keytool for creating and managing KeyStores.

### Why use it?

* Security: Protects keys and certificates from unauthorized access.   
* Organization: Provides a central place to manage cryptographic information.   
* Portability: Easy to move, copy, or share.

### Common uses:

* Secure communication over HTTPS (SSL/TLS).   
* Digitally signing documents and code.   
* Verifying the identity of others.   

## How to create Java KeyStore

The keytool command is run in a openjdk:11 image Docker container.

```shell
DN="CN=MyName, OU=MyOrgUnit, O=MyOrg, L=MyCity, ST=MyState, C=MyCountry"

docker run --rm -v $(pwd)/artifacts:/app -w /app openjdk:11 keytool -genkeypair -alias mykey -keyalg RSA -keystore keystore.jks -keysize 2048 -storepass changeit -keypass changeit -dname "$DN"
```

*Note that you can't create an empty Java KeyStore. You need to create it with 1 key entry.*

Breakdown:

* genkeypair: Generates a key pair (a public key and a private key).
* alias mykey: Specifies an alias for the key entry.
* keyalg RSA: Specifies the algorithm for the key (RSA in this case).
* keystore keystore.jks: Specifies the name of the keystore file to be created.
* keysize 2048: Specifies the size of the key.
* storepass changeit: Specifies the password for the keystore.
* keypass changeit: Specifies the password for the key.
* dname "$DN": Uses the DN variable for the Distinguished Name fields.

After running this script, a new keystore file named keystore.jks will be created. To inspect it run the following:

```shell
docker run --rm -v $(pwd)/artifacts:/app -w /app openjdk:11 keytool -list -keystore keystore.jks -storepass changeit
```

## Create a Certificate Signing Request (CSR)

A Certificate Signing Request (CSR) is a file that you create when you want to get a digital certificate from a Certificate Authority (CA). It contains information about your organization and your public key. You send this file to the CA (we use expisoft at Kivra), and they use it to create a certificate that verifies your identity. 

```shell
SUBJ="/C=US/O=Example Corp/CN=example.com/serialNumber=1234567890"
openssl req -new -newkey rsa:2048 -nodes -keyout artifacts/example.com.key -out artifacts/example.com.csr -subj "$SUBJ"
```

Here is a breakdown of the above:

* openssl req: Command to create and process certificate requests.
* new: Create a new certificate request.
* newkey rsa:2048: Generate a new RSA key with a length of 2048 bits. Includes the public and private key. Public key is included in the csr generated (example.com.csr).
* nodes: Do not encrypt the private key.
* keyout example.com.key: Output the private key to example.com.key.
* out example.com.csr: Output the CSR to example.com.csr.
* subj "/C=US/O=Example Corp/CN=example.com/serialNumber=1234567890": Specify the subject details directly on the command line.


## Create self signed certificate

A self-signed certificate is a certificate that you create and sign yourself, rather than getting it from a CA. It can be used for testing or internal purposes, but it won't be trusted by browsers or other systems that rely on CA-issued certificates.

```shell
openssl x509 -req -in artifacts/example.com.csr -signkey artifacts/example.com.key -out artifacts/example.com.crt -days 365
```

Breakdown:

* openssl x509: Command to manage X.509 certificates.
* req: Input is a certificate request.
* in example.com.csr: Input file is the CSR.
* signkey example.com.key: Sign the certificate with the private key.
* out example.com.crt: Output the self-signed certificate to example.com.crt.
* days 365: Certificate is valid for 365 days.

## Import self signed certificate into Java KeyStore

```shell
docker run -it --rm -v $(pwd)/artifacts:/app -w /app openjdk:11 keytool -importcert -alias myca -file example.com.crt -keystore keystore.jks -storepass changeit
```
Breakdown:

* keytool -importcert: Command to import a certificate into a keystore or trust store.
* alias myca: Specifies an alias for the CA certificate entry in the trust store.
* file ca-cert.crt: Specifies the CA certificate file to be imported.
* keystore mytruststore.jks: Specifies the trust store file into which the CA certificate will be imported.
* storepass changeit: Specifies the password for the trust store.

You can now see the certificate in the Java KeyStore.

```shell
docker run --rm -v $(pwd)/artifacts:/app -w /app openjdk:11 keytool -list -keystore keystore.jks -storepass changeit
```
