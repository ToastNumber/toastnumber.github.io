---
layout: post
title:  "Mutual TLS"
permalink: /mutual-tls.html
tags: java openssl tls security keystore
---

A demonstration of how to setup mutual TLS for a simple Java server.

<!--more-->

See my [mTLS](https://github.com/ToastNumber/mTLS) repo for scirpts and source code.

In mutual TLS, the TLS handshake includes a "Request CERT" message from the server to the client, which asks the client to send _its_ certificate for authentication so each side of the connection can verify the identity of the other (usually only the client can verify the server's identity).

A TCP handshake consists of three messages: syn, syn+ack and ack. A **TLS** handshake consists of the same messages plus additional message to established an encrypted connection. Let's look at (the main bits of) a cURL output:

```
$ curl -v https://www.google.com

...
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=www.google.com
*  start date: Dec  8 22:50:34 2021 GMT
*  expire date: Mar  2 22:50:33 2022 GMT
*  subjectAltName: host "www.google.com" matched cert's "www.google.com"
*  issuer: C=US; O=Google Trust Services LLC; CN=GTS CA 1C3
*  SSL certificate verify ok.
...
```

From the above, we can see the server sends us _its_ certificate but we never send a certificate of our own.

Main steps in configuring mutual TLS:

* Create a self-signed root certificate, trusted by all parties
* Create keys and certificate requests for all parties
* Sign the certificate requests with the root certificate
* Configure the server to send a "Request CERT" message to the client in the TLS handshake

## Creating the root certificate

Both sides of the connection need to verify certificates using a certificate authority (CA). For this demo I configured a private CA using `openssl req`, which is a "self-signed" certificate which I make the server/client trust later on. Then 

The [gen_ca_cert.sh](https://github.com/ToastNumber/mTLS/blob/master/security/gen_ca_cert.sh) script does this. The main command is:
```bash
openssl req \
    -newkey rsa:2048 \
    -x509 \
    -keyout cakey.pem \
    -out cacert.pem \
    -nodes \
    -subj "/CN=mtls-demo-root"
```

In plain English: "create a 2048-bit RSA key (newkey); output a self-signed certificate (x509); the subject name/issuer name of the CA is 'CN=mtls-demo-root' (subj); save the key in cakey.pem (keyout); and the certificate in cacert.pem (out)".

## Creating private keys and certificate requests

Use the `genrsa` command to create private keys:
```bash
openssl genrsa -out server.pem 2048
```

Then use the `req` command to create a certificate _request_ (you can think of it as a certificate that hasn't been signed by the CA yet). Pass in the key we just created and specify the subject name of the holder of this certificate ("OU" = "organizational unit"). It's important that the common name (CN) matches the actual hostname we use later on: when we try to connect to "https://localhost", the client will check that the server's CN matches the domain we meant to connect to in the first place, e.g. "curl: (60) SSL: certificate subject name 'test' does not match target host name 'localhost'".
```bash
openssl req -new \
	-key server.pem \
	-out server-req.pem \
	-subj "/OU=server/CN=localhost"
```

Do the above for both the server and the client so we have 4 files in total:
* server.pem/client.pem: the private keys
* server-req.pem/client-req.pem: the certificate requests


## Sign the certificate requests

The `openssl x509` is the "certificate display and signing utility". We can sign our certificate requests as follows:

```bash
openssl x509 -req \
	-in server-req.pem \
	-CA cacert.pem \
	-CAkey cakey.pem \
	-CAcreateserial \
	-out server-signed.pem
```

In plain English: "expect a certificate request (req) in server-req.pem (in); the CA's certificate is in cacert.pem (CA); the CA's private key is in cakey.pem (CAkey); create a serial number file if it doesn't already exist (CAcreateserial); output to server-signed.pem (out)."

Do the above for both the server and the client so we have 2 signed certificates: server-signed.pem and client-signed.pem.

We can inspect these certificates to understand what's happened. Using `openssl verify` we can see the chain of certificates up to the issuer:
```bash
$ openssl verify -CAfile cacert.pem -show_chain server-signed.pem
server-signed.pem: OK
Chain:
depth=0: OU = server, CN = localhost (untrusted)
depth=1: CN = mtls-demo-root
```

And `openssl x509` shows the contents of the certificate:
```bash
$ openssl x509 -noout -text -in server-signed.pem
Certificate:
    Data:
        Serial Number:
            49:af:e1:e3:93:3c:5c:b1:0c:2f:92:eb:29:a5:6c:a5:c1:19:95:49
        Issuer: CN = mtls-demo-root
        Validity
            Not Before: Jan 15 23:15:55 2022 GMT
            Not After : Feb 14 23:15:55 2022 GMT
        Subject: OU = server, CN = localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
...
```

It includes:
* A per-issuer unique serial number
* The issuer subject name (`CN = mtls-demo-root`)
* The validity window
* The subject of the certificate itself (`OU = server, CN = localhost`)
* The 2048-bit public key (omitted)

## Creating keystores

For our Java application we need _keystores_ so we need to convert our existing certificates/private keys. We can do this by saving the server's, for example, certificate and private key into a single `server.p12` file with `openssl pkcs12` and then import this into a keystore.

```bash
openssl pkcs12 -export \
    -in server-signed.pem \
    -inkey server.pem \
    -out server.p12 \
    -name server
keytool -importkeystore \
    -srcstorepass '' \
    -srckeystore server.p12 \
    -keystore server.jks \
    -storepass password
```

_Note: all of the steps in this post could have been done with `keytool` in the first place but I wanted to learn more about `openssl` ..._

## Configuring a Java server for mutual TLS

Since the client always expects the server to send its certificate, we just need to configure the server to ask the _client_ to send its certificate.

1. Set the `javax.net.ssl.trustStore` VM argument to the path of the keystore containing the CA certificate, so the application knows to verify certificates with our private CA
2. Set the `javax.net.ssl.keyStore` VM argument to the path of the keystore containing the server's certificate and private key 
3. Set the `javax.net.ssl.trustStorePassword` and `javax.net.ssl.keyStorePassword` so the server can open the truststore and keystore.
4. Configure the `SSLServerSocket` to request a certificate from the client by running `setNeedClientAuth(true)`

See a full example in [`Server.java`](https://github.com/ToastNumber/mTLS/blob/master/java/Server.java).

## Running the demo

You can see mutual TLS in action by first running the server (`./run.sh server`) and then running the client, either by using cURL (see below) or `.run.sh client`.

**No CA**: if we try to connect without even specifying the CA certificate, the client can't verify the server's certificate so we would get an error.
```bash
$ curl https://localhost:5000
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

**No certificate**: if we try to connect without specifying a valid client certificate then the server can't verify the client so the handshake fails. **This proves we've achieved one of the key objectives of mutual TLS**. Using the `--cacert` argument we can verify the server's certificate but the client will send an empty certificate chain and so the server throws an exception: `javax.net.ssl.SSLHandshakeException: Empty client certificate chain`.
```bash
curl --cacert cacert.pem https://localhost:5000
curl: (56) OpenSSL SSL_read: error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate, errno 0
```

**CA, certificate and key**: when the client is configured with the CA certificate, its own certificate and private key, then it can verify the server and the server can verify the client!  The `--cert` argument is the client's certificate and the  `--key` argument is the client's private key:

```bash
curl --cacert cacert.pem \
    --cert client-signed.pem \
    --key client.pem \
    https://localhost:5000
<h1>Hello, World!</h1>
```


