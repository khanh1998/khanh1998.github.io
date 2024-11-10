---
date: '2022-06-05T20:01:30+07:00'
draft: false
title: 'A note in SSL/TLS'
tag: ['networking']
---
## What is SSL? and why do we need them?
SSL is a method to encrypt your data before transmitting it to the receiver. If we don’t encrypt and send data as plain text, somebody can sniff in your connection to read all your messages.

## How do SSL works?
Of course, to establish an SSL, we need a TCP connection first, let’s assume that we have a TCP connection ready.

![](ssl-connection.png 'How an SSL connection is established')

1. The client says hello to the server by sending a request with the below data:
    - TLS version client supports (1.0 2.0 ..).
    - cipher suites client supports (algorithm).
    - a random byte: client random.
2. The server says hello to the client by sending a request with the below data:
    - Server’s SSL certificate.
    - the server’s chosen cipher suites.
    - a random byte: server random.
3. The client verifies the SSL certification of the server against its list of Certificate Authorities (CA)
    - generate a pre-master secret
    - encrypted the secret with the server public key and sent it back to the server
4. Server using its the private key to decrypt the pre-master secret using the agreed cipher.
    - Both server and client now can generate the master secret based on client random, server random and pre-master secret.
5. From now on, data transmitting between server and client will be encrypted symmetrically using master secret.

Read more about the [SSL handshake](https://www.acunetix.com/blog/articles/establishing-tls-ssl-connection-part-5/).

## Can SSL be faked?
An SSL usually contains some information like owner/organization, its location public key, validity dates,…

For example, if a hacker wants to fake a Google service, he will make a UI look exactly like Google, with the URL slightly different. Since Google adopt SSL so he wants to fake an SSL certificate as well:
1. Generate a new pair of the public and private key
2. Generate a new SSL cert that has information that looks exactly like the one of Google

How do they prevent that? That is when [CA – Certificate Authority](https://www.techtarget.com/searchsecurity/definition/certificate-authority) comes into play. To generate an SSL cert is quite simple, but that is not enough, you need to register your SSL cert with a CA as well, and they make sure you can’t register a fake URL.

This situation is similar to someone trying to fake me, my name is Khanh and he claims that his name is Khanh too? So how do others differentiate me and the fake one? In real life, we have an organization that issues ID cards, each citizen has a unique ID, if you want to verify where I am the real Khanh, then take the ID and verify with the ID issuing organization because we trust them.

So CA is basically a third party that we trust at them, and an SSL is like a citizen ID in real life. They also have paired public and private keys. Conceptually, your certification will be encrypted by their private key, and their public is installed in your browser and OS.

Every computer comes with a set of CA public keys out of the box. When the client receives the SSL cert from the server, they will decode the cert using the public key of CA, if it can, then it means the Cert is valid.

## Replay attack
![](replay-attack.png 'Replay Attack')
Do you wonder why the master secret – the symmetric key used for encrypted data – is made of: client random + server random + pre-master secret?

When we use only pre-master secret as a master secret:
1. Man in the middle (MiM) can catch your packet, but he can’t decrypt your data.
2. He will catch all of your packets and resend them from begging of the SSL handshake, event client hello,…
3. That lead to the hacker now will have another connection that has the same master secret as the user
4. The hacker can’t know the secret and the information but he doesn’t need it.
5. He catches your packet and resends it to the server, it is able to decrypt the message since it has the same master secret.

So what can the hacker do to make some damage? For example, you order one cup of coffee, but he can resend your request 10 more times, and now you have 11 cups of coffee. Or resend the login request to get an authentication token.

How to prevent that?

In SSL, the server will assign and send to each client a random unique value. If the hacker catch and resend every one of your packets since the beginging of the SSL handshake, he will have a connection, but the server random value is different (step server hello). The hacker cannot take a packet of the user and resend it after that because the server randomly is not the same as the user.

## Why don’t we use asymmetric keys to exchange data?
Each server has a pair of public and private keys, why don’t the client just encrypt data with the public key, and the server can decrypt data using the private key. Vice versa, the server encrypts data with a private key, and the client decrypts using the public key. Those are based on characteristics of an asymmetric key.

Because encrypting data using a symmetric key (the master secret) is faster and needs fewer resources when compared to an asymmetric key.

## References
https://www.ibm.com/docs/en/cics-tg-multi/9.2?topic=ssl-how-connection-is-established
