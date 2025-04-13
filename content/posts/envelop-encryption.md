---
date: '2025-04-11T20:56:56+07:00'
draft: false
tags: ['cs', 'encryption']
title: 'Envelop Encryption'
---

To secure storing sensitive information, a common approach is to use **envelope encryption**:
- A **master key** (usually managed by a secure KMS)
- A **row-level key** (used to encrypt individual records)
- Each row of sensitive data (e.g., credit card information) is encrypted using a unique row key, the row key itself is encrypted using the master key and stored alongside the data in the database.

This setup allows secure key rotation, better isolation, and compliance with security standards.

## 1. Data Encryption
- For an application, generate a single master key (KEK - Key Encryption Key) and store it in a safe place
- For each row (e.g., credit card), we generate a unique key (DEK - Data Encryption Key)
- We encrypt row data with its corresponding row key and also encrypt the row key with the master key
    - we usually don't need to encrypt the entire row of data, just some sensitive fields are enough
    - for the credit card use case, we only need to encrypt the card number, cardholder name, and CVV
    - this keeps the encryption small and fast
- Store encrypted data and encrypted row key to the database
## 2. Data Decryption
- Retrieve encrypted data and encrypted row key from the database
- Decrypt row key with master key
- Decrypt row data with row key
## 3. Where To Store Master Key (KEK)
1. KMS - Key Management Service
    - The safest place to manage the master key
    - KMS is backed by a Hardware Security Module (HSM). HSM is special hardware that helps you to generate and store keys, and encrypt and decrypt data with the generated key. The generated keys will never leave the device, no one knows the key, even the administrators, it also usually comes with data tampering prevention out of the box.
    - Once the master key is created by KMS, you can send row keys to KMS to encrypt it. To decrypt row keys, send the encrypted row keys back to KMS, which will return the decrypted ones. The master key never leaves the HSM.
    - Major cloud providers all support this service
    - If you use KMS from the cloud providers, your master keys are generated and stored securely within their infrastructure. You can even gain more control over security with BYOE - [Bring Your Own Encryption](https://aws.amazon.com/blogs/security/are-kms-custom-key-stores-right-for-you/), keys now will be generated and stored in your custom HSM devices.
    - If you need absolute security control, host your own KMS service that is backed by HSM.
2. Secrets Manager
    - Not as safe as KMS
    - This service is meant to store API keys, passwords, certificates, and other sensitive data - not master keys (KEK)
    - Since it isn't backed by HSM, hence it doesn’t support key generation, storage, or cryptographic operations like encryption and decryption.
    - To encrypt or decrypt the row key, your application has to fetch the master key from the secret manager and load it to memory. Therefore, your master key can be leaked.
    - Major cloud providers also support this service
3. Environment variable
    - Least secure option, not recommended
    - Any process or thread in your app can get the master key, 
    - They may appear in crash logs or dump files, posing a security risk
    - Visible via `/proc/<pid>/environ`
    - Not Rotatable or Audited

## 4. Decryption Performance Concerns
1. KMS - Key Management Service
    - Decrypting a row key requires a network call to KMS, which may introduce latency
    - You can mitigate this issue by securely caching the decrypted row key in memory to reuse it
## 5. Master Key Rotation
Master key rotation can help to boost your security level. This process requires you to decrypt row keys with the old master key and re-encrypt them with a new master key. Row data remains intact since we don't change the row key.
- Since this process can take time, so you might need to support multiple versions of the master key during rotation.
- Consider batch job or lazy approach to enhance the performance.
## 6. Additional Securing Credit Card Info Approach
If you don't have to store user credit card information, consider using a Payment Service Provider like Stripe, which handles the credit card information on your behalf.
## 7. KMS is great, so why not just use it to encrypt everything?
- Here is the explanation from AWS:
    > While AWS KMS does support sending data up to 4 KB to be encrypted directly, envelope encryption can offer significant performance benefits. When you encrypt data directly with AWS KMS it must be transferred over the network. Envelope encryption reduces the network load since only the request and delivery of the much smaller data key go over the network. The data key is used locally in your application or encrypting AWS service, avoiding the need to send the entire block of data to AWS KMS and suffer network latency.
    > https://aws.amazon.com/kms/faqs/
    So basically, the row key is small and safe to transmit over the network.

- Some cloud providers like AWS will throttle your requests when you exceed the **default** [quota](https://docs.aws.amazon.com/kms/latest/developerguide/requests-per-second.html), like 1000 requests per second. Be aware of this, since it can become the bottleneck, and affects your application scalability.
- HSMs are usually limited to some operations and specific algorithms. Using the row key allows you to choose encryption algorithms that best fit your application's needs.
## 7. Why is it safe
- If you manage your master key in a proper KMS, then the chance your master key gets leaked is very low, you can also enhance master key security by doing master key rotation.
- The row key is more vulnerable since it must be loaded into memory by your application to decrypt data. If improper actions are taken, like showing sensitive data in system logs, it could be leaked.
- But if your row keys are leaked, then only the corresponding data with those leaked keys are leaked, not all since each record is protected by its own key.

## 8. References
- https://aws.amazon.com/kms/faqs/
- https://docs.aws.amazon.com/kms/latest/developerguide/requests-per-second.html
- https://aws.amazon.com/blogs/security/are-kms-custom-key-stores-right-for-you