---
date: '2024-11-09T20:12:19+07:00'
draft: true
title: 'ACID'
tags: ['database']
---
**Atomicity**: Each transaction could have multiple steps in it, each step might be a query or an update to the data. Atomicity means that the transaction is considered a success if all steps are a success. If just one of those steps is failed, the whole transaction is considered to fail, and it has to roll back to the previous state.

**Consistency**: In the database, it might have some rules or constraints, and the transaction when modified data have to follow those constraints. For example, let say the constrain for bank account balance is 0 or positive, so if a transaction tries to assign a negative number to a bank account, it violates the constrain and has to roll back.

**Isolation**: At a time, there could be multiple transactions are trying to read or write to the database concurrently. There is nothing to say if all transactions are not trying to access or modify the same data, but in reality, they do a lot. So, the goal of isolation is to ensure that executing multiple transactions at the same time does not lead to a consistent state. They usually use a lock to achieve isolation in the database.

**Durability**: This may be the easiest term to understand in ACID. Durability means once I have committed my transaction, my data will be in the database whenever I need it. The server could be down for some reason, but when it comes to online again, my data have to be ready to serve.
