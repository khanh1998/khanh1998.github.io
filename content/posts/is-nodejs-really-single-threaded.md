---
date: '2022-05-28T19:18:41+07:00'
draft: false
title: 'Is NodeJS really single-threaded?'
tags: ['nodejs','os']
---
No! Not really!

NodeJS has one main thread called event-loop and it also has a worker pool to handle blocking IO and CPU-intensive tasks. The default number of threads in the worker pool is 4, and you can change it.

>*Node.js uses a small number of threads to handle many clients. In Node.js there are two types of threads: one Event Loop (aka the main loop, main thread, event thread, etc.), and a pool of k Workers in a Worker Pool (aka the threadpool).*
>
> -- **Don’t Block the Event Loop (or the Worker Pool)**

You can test it by yourself. Firstly, run the below code snippet, it gonna log out the PID – process id of the app.

```javascript
const http = require('http');
const hostname = '127.0.0.1';const port = 3000;
const server = http.createServer(
    (req, res) => {
        res.statusCode = 200;
        res.setHeader('Content-Type', 'text/plain');
        res.end('Hello World');
    }
);
server.listen(port, hostname, () => { console.log(`PID: ${process.pid}`);});
```
Next, execute the below command in the Linux terminal, it gonna print out the number of threads of a PID.
```sh
ps huH p <PID> | wc -lv
```
For MacOS:
```sh
ps -M PID | grep -v USER | wc -l
```
On my computer, the number of threads is 7. The number of threads varies according to different machines, but my point is that NodeJS use more than one thread.

## References
https://kariera.future-processing.pl/blog/on-problems-with-threads-in-node-js/
https://stackoverflow.com/questions/61550822/why-node-js-spins-7-threads-per-process
