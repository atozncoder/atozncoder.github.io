---
title: Netty SSL/TLS with CA signed certificates
description: Netty SSL/TLS with CA(trusted certificate authority) signed certificates.
date: 2020-10-01
tags:
  - posts-tag
layout: layouts/post.njk
---

## Server certificate
In the description of the [TLS](https://tools.ietf.org/html/rfc5246) protocol for [Server certificate](https://tools.ietf.org/html/rfc5246#page-47) it states that it's structure is a sequence (chain) of certificates where the servers's certificate MUST come first in the list. Each following certificate MUST directly certify the one preceding it. Image below taken from the [Browsers and Certificate Validation](https://www.ssl.com/article/browsers-and-certificate-validation/) shows this in more details:

![cert_chain](/img/casignednettysecuredchat/certification_paths.png)



## Create the certificates
Process to create certificates that can be used for test application can be found [here](https://help.f-secure.com/product.html#business/threatshield/latest/en/concept_E8E015C30E05412190F22C5DFC36AC0B-threatshield-latest-en). For our example just following steps are needed:
* [Create a self-signed root CA certificate](https://help.f-secure.com/product.html#business/threatshield/latest/en/task_50407934989D4923AB76367EA0E627CA-threatshield-latest-en)
* [Create an intermediate CA certificate for creating other certificates](https://help.f-secure.com/product.html#business/threatshield/latest/en/task_9EF132D90B9241268DF4EC8CA5FADBBF-threatshield-latest-en)
* [Create a leaf certificate for the webserver of the web UI](https://help.f-secure.com/product.html#business/threatshield/latest/en/task_D81B8959CD3643C5A9E8DD0E2A4EF32E-threatshield-latest-en)

Please note that for real production system real CA cert is needed and not self signed one.

## Configure Server SSL context
Once certs are created, we need to setup our application to use them. For server we need following files:
* server.key - Server public key.
* server.pem - Server private key.

If we look our example folder structure, this is how it can be setup:

![server_certs_folder_strucutre](/img/casignednettysecuredchat/server_certs_folder_strucutre.png)

Now if we configure path to the certs in our config file:
```
server.cert.path=/certs/server/server.pem
server.key.path=/certs/server/server.key
```
**Note**: I am using [OWNER](http://owner.aeonbits.org/) for configuration. This is not required for netty SSL.

Then the following code can be used to load SSL context for server:
```
InputStream keyCertChainIS = getClass().getResourceAsStream(cfg.serverCertPath());
InputStream keyIS = getClass().getResourceAsStream(cfg.serverKeyPath());
sslCtx = SslContextBuilder.forServer(keyCertChainIS, keyIS).build();
```

## Configure Client SSL context
We need to import our root CA to be trusted root CA to our client. We need:
* rootCA.crt - Root CA cert file

In our example it is setup as follows:
![client_ca_cert_folder_structure](/img/casignednettysecuredchat/client_ca_cert_folder_structure.png)

And we load it with following code:
```
final SslContext sslCtx = SslContextBuilder.forClient()
        .trustManager(getInputStream())
        .build();
```
where we get the input stream as follows:
```
if ("classpath".equalsIgnoreCase(cfg.sslStreamReader())) {
    return getClass().getResourceAsStream(cfg.clientCertPath());
} else {
    return new FileInputStream(cfg.clientCertPath());
}
```
and we have path to cert in classpath configured as follows:
```
client.cert.path=/certs/client/rootCA.crt
ssl.stream.reader=classpath
```

## Complete example and demo
The complete sample code can be found here:https://github.com/atozncoder/netty_secured_chat

It can be tested by performing following steps:
```
mvn clean package
```

In one terminal start the server:
```
java -jar target/netty_seccured_chat-1.0-SNAPSHOT-jar-with-dependencies.jar
```
In another terminal start a client:
```
java -jar target/netty_seccured_chat-1.0-SNAPSHOT-jar-with-dependencies.jar client
```
