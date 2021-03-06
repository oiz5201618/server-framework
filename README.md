# Server Framework in Modern C

* [`async`](async.h): A native POSIX thread pool.
  - It uses a combination of a pipe (for wakeup signals) and
    mutexes (for managing the task queue).
  - Give a basic layer of protection to any server implementation.
* [`reactor`](reactor.h): Reactor pattern implementation using callbacks
  - Linux epoll system call abstraction
  - Supported events: ready to read/write, closed, reactor exit
* [`protocol-server`](protocol-server.h) - server construction library
  - protocol-server manages everything that makes a server run, including
    the thread pool, process forking, accepting new connections, setting up
    the initial protocol for new connections, and user space socket writing
    buffers.

Using this framework to build your own services is simple to use. It is
based on Protocol structure and callbacks, so that we can dynamically change
protocols and support stuff such as HTTP requests.

Here is a simple example:
```c
#include "protocol-server.h"
#include <stdio.h>
#include <string.h>

void on_open(server_pt server, int sockfd) {
    printf("A connection was accepted on socket #%d.\n", sockfd);
}

void on_close(server_pt server, int sockfd) {
    printf("Socket #%d is now disconnected.\n", sockfd);
}

// simple echo implementation: the main callback
void on_data(server_pt server, int sockfd) {
    // We will assign a reading buffer on the stack
    char buff[1024];
    ssize_t incoming = 0;
    // Read everything, this is edge triggered, `on_data` won't be called
    //again until all the data was read.
    while ((incoming = Server.read(server, sockfd, buff, 1024)) > 0) {
        // since the data is stack allocated, we'll write a copy
        // otherwise, we would avoid a copy using Server.write_move
        Server.write(server, sockfd, buff, incoming);
        if (!memcmp(buff, "bye", 3)) {
            // close the connection automatically AFTER the buffer was sent.
            Server.close(server, sockfd);
        }
    }
}

int main(void) {
  /* create the echo protocol object */
  struct Protocol protocol = { .on_open = on_open,
                               .on_close = on_close,
                               .on_data = on_data,
                               .service = "echo"};
  // use the macro start_server, which will call Server.listen(settings)
  // with the settings we provide.
  start_server(.protocol = &protocol, .timeout = 10, .threads = 8);
}
```
