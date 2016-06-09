A sample created to demonstrate a 100% CPU usage issue using NodeJS clusters and mariasql.

## Directions for setup

1. Clone the repo - `git clone https://github.com/bsurendrakumar/node-simplex.git`
2. Run the SQL scripts under the sql folder - `db.sql`.
3. Set the database configuration in - `/app/db-config.js`.
4. Run `npm install`
5. Run `npm start`

## Reproducing the issue

Open the http://127.0.0.1:3002/api/v1/country/list in your browser as soon as you start your server, mostly within 120 seconds.

The reason its 120 seconds is because the generic-pool `idleTimeout` is set to 120 seconds.

> This URL will vary based on the where you're server is running.

## Why do we think its related to our usage of the mariasql library?

[Link to module](https://github.com/mscdex/node-mariasql)

Since the `node-mariasql` library does not support pooling, we are using the third party - [generic-pool]() to maintain a pool of connections. The minimum number of connections is set to 5. All its configuration can be found under `app/db-config.js`. So when the server starts, generic pool will kick of 5 connections to MySQL and keep them in its pool.

The idleTimeout for an object in the generic-pool has been set to 120 seconds. This means that if there are more than 5 objects in the pool and one of them has not been used for the last 120 seconds, it'll be destroyed.

At server startup, we're making a simple call to our country controller to fetch the list of countries. This code is [here](https://github.com/bsurendrakumar/node-simplex/blob/master/app/code/api.js#L25). This establishes a new connection to the database, so now in the pool there'll be a 6 SQL connection in the pool. This will get cleaned after 120 seconds.

Following is the step by step process following which, we believe that the issue is with our usage of the mariasql library -

- When the server is started, we are logging the process ID to the console. Grab the mater process ID, for example - **20584**.
- Look at the file descriptors being used by the process by using - `ls -l /proc/20584/fd`. Make a note of the socket connections. The output of this will look something like this -
```
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 12 -> socket:[2469914]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 13 -> socket:[2469917]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 14 -> socket:[2468106]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 15 -> socket:[2468109]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 17 -> socket:[2467206]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 18 -> socket:[2467208]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 19 -> socket:[2467210]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 2 -> /dev/tty
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 20 -> socket:[2467212]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 21 -> socket:[2467214]
lrwx------ 1 abijeet abijeet 64 Jun  9 19:24 22 -> socket:[2467306]
```
- Copy one of the sockets numbers for example *2467212*, and run `lsof | grep 2467212`. You'll notice that these are connections to the MySQL server. The output of that should be something like -
```
node      20584           abijeet   20u     IPv4            2467212       0t0        TCP localhost:57092->localhost:mysql (ESTABLISHED)
V8        20584 20585     abijeet   20u     IPv4            2467212       0t0        TCP localhost:57092->localhost:mysql (ESTABLISHED)
V8        20584 20586     abijeet   20u     IPv4            2467212       0t0        TCP localhost:57092->localhost:mysql (ESTABLISHED)
V8        20584 20587     abijeet   20u     IPv4            2467212       0t0        TCP localhost:57092->localhost:mysql (ESTABLISHED)
V8        20584 20588     abijeet   20u     IPv4            2467212       0t0        TCP localhost:57092->localhost:mysql (ESTABLISHED)
```
- Crash the server by going to http://127.0.0.1:3002/api/v1/country/list
- Wait for the MySQL connection in the master thread to be closed. This is logged to the console -
```
Destroying / ending master thread ID -  4984
```
- Next run, `strace -o log.txt -eepoll_ctl,epoll_wait -p 20584`. Note that you might need to install **strace**. This command logs all the `epoll_ctl, epoll_wait` system calls made by the Node.JS process and puts it inside the current working directory.
- Open the log.txt file, and you'll notice the following logs -
```
epoll_wait(5, {{EPOLLIN|EPOLLHUP, {u32=16, u64=16}}}, 1024, 847) = 1
epoll_ctl(5, EPOLL_CTL_DEL, 16, 7ffe441aa850) = -1 EBADF (Bad file descriptor)
epoll_wait(5, {{EPOLLIN|EPOLLHUP, {u32=16, u64=16}}}, 1024, 845) = 1
epoll_ctl(5, EPOLL_CTL_DEL, 16, 7ffe441aa850) = -1 EBADF (Bad file descriptor)
epoll_wait(5, {{EPOLLIN|EPOLLHUP, {u32=16, u64=16}}}, 1024, 843) = 1
epoll_ctl(5, EPOLL_CTL_DEL, 16, 7ffe441aa850) = -1 EBADF (Bad file descriptor)
```
- The file descriptor here is **16**, and if you co-relate it with your earlier `ls -l /proc/20584/fd` and `lsof | grep 2467212`, you'll realize that this belongs to the MySQL connection that was just closed.