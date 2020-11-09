---
# title: Netty reloading SSL/TLS certificate.
title: draft
description: Reloading SSL certificate without restarting the server.
date: 2020-10-01
tags:
  - posts-tag
layout: layouts/post.njk
---

Security is hot topic these days. Data has to be protected. Regulations have to be followed. Encrypted is the buzzword and more often than not SSL/TLS is a way to do it. 

Setting SSL/TLS implies having valid CA signed certificate on the server.  Please check the previous post, [Netty SSL with CA signed certificates](/posts/casignednettysecuredchat/), for explanations on how to create SSL certificates and used them to run netty secured chat example. 

In addition more often then not security standards require for certificates to be reloaded periodically. To solve this some basic understanding of TLS handshake and Netty channel bootstrapping is required.

## TLS/SSL handshake
TLS use symetric encription to encrypt and decrypt traffic between client and server. When nagotiating the TLS session at some point client and server create the session key and use it for symetric encryption.


More details on how this handshake process works can be found in [What is a TLS handshake?](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/) article on Cloudflare. 

We will skip the details here, but the important thing to note is the step #6 from mentioned article. This step states: 
**6. Session keys created**: *Both client and server generate session keys from the client random, the server random, and the premaster secret. They should arrive at the same results.*

What this means is that certificate is only used for handshake and for cration of session key in process of establishing the session. Each new session will create new session key.

Based on the above we can repleace our server certificate at any time. Client that is already connected to server will not lose connection as it already nagotiated session key. Later clients can handshake with new key loaded in the meantime. Both clients should be able to stay connected, regardles to the server cert used for TLS handshake.

Following diagram shows two clients connecting to server with TLS cert reloaded:
![tls_reload](/img/nettyreloadablesslcertificate/tls_reload.png)

## Netty Server Bootstraping
Netty server bootstraping involves binding a process to a given port for each client that requests a connection. Each time a requests for a connection is received, netty server creates a channel and binds a port. In this process and when SSL is enabled, netty will load SSL engine. We can plugin our implementatio of `SslContext` here and take over the cert load process from disk or wherever the cert files are stored.



## Implementation
Implementation of the context is quite simple. Below is the important code that does what we need:
```
public class ReloadableSslContext extends SslContext {
    private SslContext sslContext;
    private SSLEngine sslEngine;

    public ReloadableSslContext() throws SSLException {
        Configuration cfg = ConfigFactory.create(Configuration.class);
        loadContext(cfg.serverCertPath(), cfg.serverKeyPath());
    }

    private void loadContext(String certPath, String keyPath) throws SSLException {
        InputStream keyCertChainIS = getClass().getResourceAsStream(certPath);
        InputStream keyIS =getClass().getResourceAsStream(keyPath);
        sslContext = SslContextBuilder.forServer(keyCertChainIS, keyIS).build();
    }

    public void reload(String certPath, String keyPath) throws SSLException {
        loadContext(certPath, keyPath);
    }
    .
    .
    .
```
**Note**: This code is not thread safe and is only shown as an example of design approach and not as production ready code.

## Usage
Reloadable SSL context can be used when creating server channel same as any other SSLContext implementation.
Code example:
```
public void start() throws InterruptedException, SSLException {

    Configuration cfg = ConfigFactory.create(Configuration.class);

    sslCtx = new ReloadableSslContext();

    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new SecureChatServerInitializer(sslCtx));

        bootstrap.bind(cfg.port()).sync().channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```

And then the secure channel server initializer uses it as follows:
```
pipeline.addLast(sslCtx.newHandler(ch.alloc()));
```

## Demo

Checkout or download the sourece code from git: https://github.com/atozncoder/netty_secured_chat/tree/relodable_ssl

Use relodable_ssl branch.

In a console run following 2 commands:
```
mvn clean package
java -jar target/netty_seccured_chat-1.0-SNAPSHOT-jar-with-dependencies.jar server
```

In second console run following 2 commands:
```
java -jar target/netty_seccured_chat-1.0-SNAPSHOT-jar-with-dependencies.jar client
ssl reload
```

Note that connection message shows that server cert subject is: `Subject DN: CN=goldius.rd.netty.ssl.org`.

The `ssl reload` command will be handlen by server so that it would use diferent cert from that moment.

In third console run following 
```
java -jar target/netty_seccured_chat-1.0-SNAPSHOT-jar-with-dependencies.jar client
```

Note the connection message and that it shows that cert subject changed to: `Subject DN: CN=goldius.rd.netty_2.ssl.org`.

Test both clients connections by sending any message.