# Exercise: Socket Programming with Event loop

## Server Flow

Creation of socket-based connections requires several operations.  
First, a socket is created with `socket`.  
Second, bound to an address with `bind`. Next, a willingness to accept incoming connections and a queue limit for incoming connections are specified with `listen`.  
Finally, the connections are accepted with `accept`.

#### Accept a connection on a socket

`accept()` extracts the first connection request on the queue of pending connections, creates a new socket with the same properties of socket, and allocates a new file descriptor for the socket.  
If no pend-ing connections are present on the queue, and the socket is not marked as non-blocking, accept\(\) blocks the caller until a connection is present.

### Code

```c
/* 
 * tcpserver.c - A simple TCP echo server 
 * usage: tcpserver <port>
 */

#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h> 
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define BUFSIZE 1024

/*
 * error - wrapper for perror
 */
void error(char *msg) {
  perror(msg);
  exit(1);
}

int main(int argc, char **argv) {
  int parentfd; /* parent socket */
  int childfd; /* child socket */
  int portno; /* port to listen on */
  int clientlen; /* byte size of client’s address */
  struct sockaddr_in serveraddr; /* server's addr */
  struct sockaddr_in clientaddr; /* client addr */
  struct hostent *hostp; /* client host info */
  char buf[BUFSIZE]; /* message buffer */
  char *hostaddrp; /* dotted decimal host addr string */
  int optval; /* flag value for setsockopt */
  int n; /* message byte size */

  /* 
   * check command line arguments 
   */
  if (argc != 2) {
    fprintf(stderr, “usage: %s <port>\n”, argv[0]);
    exit(1);
  }
  portno = atoi(argv[1]);

  /* 
   * socket: create the parent socket 
   */
  parentfd = socket(AF_INET, SOCK_STREAM, 0);
  if (parentfd < 0) 
    error("ERROR opening socket");

  /* setsockopt: Handy debugging trick that lets 
   * us rerun the server immediately after we kill it; 
   * otherwise we have to wait about 20 secs. 
   * Eliminates "ERROR on binding: Address already in use" error. 
   */
  optval = 1;
  setsockopt(parentfd, SOL_SOCKET, SO_REUSEADDR, 
         (const void *)&optval , sizeof(int));

  /*
   * build the server’s Internet address
   */
  bzero((char *) &serveraddr, sizeof(serveraddr));

  /* this is an Internet address */
  serveraddr.sin_family = AF_INET;

  /* let the system figure out our IP address */
  serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);

  /* this is the port we will listen on */
  serveraddr.sin_port = htons((unsigned short)portno);

  /* 
   * bind: associate the parent socket with a port 
   */
  if (bind(parentfd, (struct sockaddr *) &serveraddr, 
       sizeof(serveraddr)) < 0) 
    error(“ERROR on binding”);

  /* 
   * listen: make this socket ready to accept connection requests 
   */
  if (listen(parentfd, 5) < 0) /* allow 5 requests to queue up */ 
    error(“ERROR on listen”);

  /* 
   * main loop: wait for a connection request, echo input line, 
   * then close connection.
   */
  clientlen = sizeof(clientaddr);
  while (1) {

    /* 
     * accept: wait for a connection request 
     */
```

## Client Flow

First, a socket is created with `socket`.  
Second, build the server’s Internet address. Next, create a connection with the server with `connect`. Finally, send the message line to the server, and print server’s reply using \(`write`, and `read`\).

### Code

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h> 

#define BUFSIZE 1024

/* 
 * error - wrapper for perror
 */
void error(char *msg) {
    perror(msg);
    exit(0);
}

struct sockaddr_in prepare_sock_addr(int portno, char *hostname) {
  struct sockaddr_in serveraddr;
  struct hostent *server;

  /* gethostbyname: get the server’s DNS entry */
  server = gethostbyname(hostname);
  if (server == NULL) {
      fprintf(stderr,”ERROR, no such host as %s\n”, hostname);
      exit(0);
  }

  /* build the server’s Internet address */
  bzero((char *) &serveraddr, sizeof(serveraddr));
  serveraddr.sin_family = AF_INET;
  bcopy((char *)server->h_addr, 
  (char *)&serveraddr.sin_addr.s_addr, server->h_length);
  serveraddr.sin_port = htons(portno);

  return serveraddr;
}

void cli_send_request(struct sockaddr_in serveraddr) {
  /* socket: create the socket */
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  if (sockfd < 0) 
      error(“ERROR opening socket”);

  /* connect: create a connection with the server */
  if (connect(sockfd, &serveraddr, sizeof(serveraddr)) < 0) 
    error(“ERROR connecting”);

  /* get message line from the user */
  char buf[BUFSIZE];
  printf("Please enter msg: “);
  bzero(buf, BUFSIZE);
  fgets(buf, BUFSIZE, stdin);

  /* send the message line to the server */
  int n = write(sockfd, buf, strlen(buf));
  if (n < 0) 
    error(“ERROR writing to socket”);

  /* print the server’s reply */
  bzero(buf, BUFSIZE);
  n = read(sockfd, buf, BUFSIZE);
  if (n < 0) 
    error(“ERROR reading from socket”);
  printf("Echo from server: %s", buf);
  close(sockfd);
}

void client_special_request(struct sockaddr_in serveraddr) {
  /* socket: create the socket */
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  if (sockfd < 0) 
      error("ERROR opening socket");

  /* connect: create a connection with the server */
  if (connect(sockfd, &serveraddr, sizeof(serveraddr)) < 0) 
    error("ERROR connecting");

  /* get message line from the user */
  char *text = "This is from client!\n";

  /* send the message line to the server */
  int n = write(sockfd, text, strlen(text));
  if (n < 0) 
    error("ERROR writing to socket");

  //printf("Wrote to server: %s", text);

  /* print the server's reply */
  char buf[BUFSIZE];
  bzero(buf, BUFSIZE);
  n = read(sockfd, buf, BUFSIZE);
  if (n < 0) 
    error("ERROR reading from socket");
  printf("Echo from server: %s", buf);
  close(sockfd);
}
```

```c
/* 
 * tcpclient.c - A simple TCP client
 * usage: tcpclient <host> <port>
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include “sock_helper.h"


int main(int argc, char **argv) {
    int portno;
    struct sockaddr_in serveraddr;

    char *hostname;

    /* check command line arguments */
    if (argc != 3) {
       fprintf(stderr,”usage: %s <hostname> <port>\n”, argv[0]);
       exit(0);
    }
    hostname = argv[1];
    portno = atoi(argv[2]);

    serveraddr = prepare_sock_addr(portno, hostname);
    cli_send_request(serveraddr);

    return 0;
}
```

## Use `select` for event loop

```c
  // bind and listen code are here

while (1) {
    FD_ZERO(&readfds);
    FD_SET(parentfd, &readfds);

    if (select(parentfd+1, &readfds, 0, 0, 0)< 0) {
      error(“Error in select”);
    }

    if (FD_ISSET(parentfd, &readfds)) {
      childfd = accept(parentfd, (struct sockaddr *) &clientaddr, &clientlen);
      …
```

## Use `kqueue` for event loop

In Mac, `kqueue` is similar to `epoll`.

```text
int kevent(int kq, const struct kevent *changelist, 
           int nchanges, struct kevent *eventlist, 
           int nevents, const struct timespec *timeout);
```

```c
  int kq = kqueue();
  struct kevent ev_set;
  EV_SET(&ev_set, parentfd, EVFILT_READ, EV_ADD, 0, 0, NULL);

  struct kevent evList[32];

  while (1) {
    printf("waiting for client to connect me…\n”);

    int nev = kevent(kq, &ev_set, 1, evList, 32, NULL);
    for (int i=0; i<nev; i++) {
      if (evList[I].ident != parentfd) {
        continue;
      }
      handle_connection(parentfd, serveraddr);
    }
  }
```

`EV_SET` adds `parentfd` to event set, and monitor when fds is ready for reading. If the result in `evList` contains the `parentfd`, we know there is a queued connection, so we start to handle.

## A multithreading TCP client

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include “sock_helper.h”

void *fire_request( void *ptr );

int main(void)
{
  pthread_t thread1, thread2, thread3;
  char *message1 = “Thread 1";
  char *message2 = “Thread 2”;

/* Create independent threads each of which will execute function */

  pthread_create( &thread1, NULL, fire_request, (void*) message1);
  pthread_create( &thread2, NULL, fire_request, (void*) message2);

  /* Wait till threads are complete before main continues. Unless we  */
  /* wait we run the risk of executing an exit which will terminate   */
  /* the process and all threads before the threads have completed.   */

  pthread_join( thread1, NULL);
  pthread_join( thread2, NULL); 

  return 0;
}

void *fire_request( void *ptr )
{
  char *hostname = "localhost";
  int portno = 5555;
  struct sockaddr_in serveraddr = prepare_sock_addr(portno, hostname);

  // Create socket, connect, write msg.
  client_special_request(serveraddr);

  return NULL;
}
```

## Makefile Example

```text
all: tcpserver tcpclient manyclients selectserver epollserver

tcpserver: tcp_server.c
    gcc -o tcpserver tcp_server.c

selectserver: select_server.c
    gcc -o selectserver select_server.c

epollserver: epoll_server.c
    gcc -o epollserver epoll_server.c

tcpclient: sock_helper.h tcp_client.c
    gcc -o tcpclient tcp_client.c

manyclients: sock_helper.h concurrent_client.c
    gcc -o manyclients -lpthread concurrent_client.c

clean:
    rm -f *.o tcpserver tcpclient manyclients selectserver epollserver
```

## References

[Linux Tutorial: POSIX Threads](https://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html%0A%0Ahttps://www.cs.cmu.edu/afs/cs/academic/class/15213-f99/www/class26/%0A%0Ahttp://eradman.com/posts/kqueue-tcp.html)

[Index of /afs/cs/academic/class/15213-f99/www/class26](https://www.cs.cmu.edu/afs/cs/academic/class/15213-f99/www/class26/)

[http://eradman.com/posts/kqueue-tcp.html](http://eradman.com/posts/kqueue-tcp.html)

