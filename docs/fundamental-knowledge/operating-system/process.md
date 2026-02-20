---
layout: default
title: Process
parent: Operating system
nav_order: 4
---

# Process
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC 
{:toc}

---

## The process
Process is an execution/ instance of a program. It has its own memory, space, data and resources.
Each process runs independently and is isolated from other processes.

The memory layout of a process is divided into 4 paths:
- Text section: executable code
- Data section: global variables
- Heap section: dynamically allocated memory
- Stack section: memory for temporary data storage when invoking functions (function parameters, return addresses, local variables)

![layout-of-process-memory](/asset/image/operating-system/layout-of-process-memory.png){: width="40%" height="auto" }

## Process state
A process can be one of the following states:
- New
- Running
- Waiting
- Ready
- Terminated

![process-state](/asset/image/operating-system/process-state.png){: width="80%" height="auto" }

## Process scheduling and context switch
When multiple processes are waiting for the CPU, the CPU scheduler will be the person who decide which process can be run.
Switching CPU from one process to another requires performancing a state save of current process and state restore of a different process.
This task is known as context switch.

## Process creation
Most operating systems identify processes based on a unique process identifier (pid).
The root parent process for all user processes is systemd (pid=1).
In traditional UNIX system, the first user process (pid=1) is init (System V init).

## Process termination
A zombie process is a process that is terminated but its information still exists in the process table (due to parent process hasn't called wait() yet).
An orphan process is a process whose parent is terminated before calling wait()

## Interprocess communication (IPC)
There are two models for exchanging data among processes:
- Shared memory
- Message passing

![process-state](/asset/image/operating-system/process-communication.png){: width="80%" height="auto" }

### Shared memory in Go
This is an example code to illustrate how to implement shared memory between process in Go. 
In fact, Go will make a syscall to kernel for requesting a shared memory region. The way of handling these syscalls is different per OS.
```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"golang.org/x/sys/unix"
)

const SHMSIZE = 1024
const sharedMemKey = 70000

func getMemShared() int {
	// create shared memory region of 1024 bytes
	sharedMemId, err := unix.SysvShmGet(sharedMemKey, SHMSIZE, 0666|unix.IPC_CREAT)
	if err != nil {
		log.Fatal("sysv shm create failed:", err)
	}

	return sharedMemId
}

func writer(mem []byte, message string) {
	copy(mem, []byte(message))
	fmt.Println("Wrote: ", message)
}

func reader(mem []byte) {
	fmt.Println("Received: ", string(mem))
}

func main() {
	sharedMemId := getMemShared()
	fmt.Println("Shared memory id: ", sharedMemId)

	// warning: sysv shared memory segments persist even after a process
	// is destroyed, so it's very important to explicitly delete it when you
	// don't need it anymore.
	defer func() {
		_, err := unix.SysvShmCtl(sharedMemId, unix.IPC_RMID, nil)
		if err != nil {
			log.Fatal(err)
		}
	}()

	// to use a shared memory region you must attach to it
	b, err := unix.SysvShmAttach(sharedMemId, 0, 0)
	if err != nil {
		log.Fatal("sysv attach failed:", err)
	}

	// you should detach from the segment when finished with it. The byte
	// slice is no longer valid after detaching
	defer func() {
		if err = unix.SysvShmDetach(b); err != nil {
			log.Fatal("sysv detach failed:", err)
		}
	}()

	switch os.Args[1] {
	case "write":
		writer(b, "hello world")
	case "read":
		reader(b)
	}

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	<-sigChan
	os.Exit(0)
}
```
![shared-memory-in-go](/asset/image/operating-system/shared-memory-in-go.png){: width="80%" height="auto" }

### Message passing
Message passing may be either block or nonblocking:
- Blocking send
- Nonblocking send
- Blocking receive
- Nonblocking receive

Messages exchange via message passing reside in a temporary queue, which can be implemented in 3 ways:
- Zero capacity
- Bounded capacity
- Unbounded capacity

The below is an example for implementing message passing via named pipe in go.
In UNIX system,Named pip supports bidirectional communication, but only allows half-duplex transmission (ie. data can only traverse in one direction at any time).
Additionally, communicating processes must reside on the same machine.

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"log"
	"os"
	"syscall"
	"time"

	"golang.org/x/sync/errgroup"
)

var pipeFile = "pipe.log"

func main() {
	os.Remove(pipeFile)
	err := syscall.Mkfifo(pipeFile, 0666)
	if err != nil {
		log.Fatal("Make named pipe file error:", err)
	}

	g, _ := errgroup.WithContext(context.Background())
	g.Go(
		func() error {
			readMessage(pipeFile)
			return nil
		},
	)
	g.Go(
		func() error {
			writeMessage(pipeFile)
			return nil
		},
	)

	g.Wait()
}

func readMessage(pipeName string) {
	file, err := os.OpenFile(pipeName, os.O_CREATE, os.ModeNamedPipe)
	if err != nil {
		log.Fatal("Open named pipe file error:", err)
	}

	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		message := scanner.Text()
		fmt.Println("Received message: ", message)
		if message == "\n" {
			break
		}
	}
}

func writeMessage(pipeName string) {
	f, err := os.OpenFile(pipeName, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0777)
	if err != nil {
		log.Fatal("error opening file:", err)
	}

	for i := 0; i < 10; i++ {
		f.WriteString(fmt.Sprintf("Message %d\n", i))
		time.Sleep(1 * time.Second)
	}

	f.WriteString("\n")
	f.Close()
}
```

![message-passing-in-go](/asset/image/operating-system/message-passing-in-go.png){: width="80%" height="auto" }

## Communication in client-server systems
### Socket
A socket is defined as an endpoint for communication. A pair of processes communicating over a network employs a pair of sockets—one for each process.
A socket is identified by an IP address concatenated with a port number. 

All ports below 1024 are considered well known and are used to implement standard services.

When a client process initiates a request for a connection, it is assigned a port by its host computer. 
This port has some arbitrary number greater than 1024. 
For example, if a client on host X with IP address 146.86.5.20 wishes to establish a connection with a web server (which is listening on port 80) at address 161.25.19.8, host X may be assigned port 1625. 
The connection will consist of a pair of sockets: (146.86.5.20:1625) on host X and (161.25.19.8:80) on the web server.

When working with socket, there are some important notes to take:

- TCP gives you a stream of bytes, not packets
- read() can give you fewer bytes than you asked for
- **Just stop trying to think about packets. The kernel will deal with packets. You will deal with byte streams.**

Below is an example for implementing socket in Go

```go
package server

import (
	"encoding/binary"
	"fmt"
	"io"
	"net"
	"runtime/debug"

	"github.com/sirupsen/logrus"

	"example/setting"
)

type server struct {
}

type Server interface {
}

func NewServer() Server {
	return &server{}
}

func (s *server) Start() error {
	listenerAddress := fmt.Sprintf("%s:%d", setting.Conf.ServerHost, setting.Conf.ServerPort)
	listener, err := net.Listen("tcp", listenerAddress)
	if err != nil {
		setting.GetLogger().WithError(err).Fatal("fail to start tcp server")
	}

	defer listener.Close()
	defer recoverFromPanic()

	setting.GetLogger().Info(fmt.Sprintf("start tcp server on %s", listenerAddress))

	for {
		conn, err := listener.Accept()
		if err != nil {
			setting.GetLogger().WithError(err).Error("fail to accept connection")
			continue
		}

		go s.handleRequest(conn)
	}
}

func recoverFromPanic() {
	if r := recover(); r != nil {
		setting.GetLogger().WithFields(
			logrus.Fields{
				"stack_trace": debug.Stack(),
				"recover_msg": r,
			},
		).Error("panic recovered")
	}
}

func (s *server) handleRequest(conn net.Conn) {
	defer conn.Close()

	for {
		headerBuffer := make([]byte, 4)
		_, err := io.ReadFull(conn, headerBuffer)
		if err == io.EOF {
			setting.GetLogger().Info("client end connection closed")
			return
		}
		if err != nil {
			setting.GetLogger().WithError(err).Error("error read header")
			return
		}

		messageSize := binary.BigEndian.Uint32(headerBuffer)
		messageBuffer := make([]byte, messageSize)
		_, err = io.ReadFull(conn, messageBuffer)
		if err != nil {
			setting.GetLogger().WithError(err).Error("error read message")
			return
		}

		conn.Write(messageBuffer)
	}
}
```

#### Open question
1. Why do we use io.ReadFull instead of conn.Read?
2. If we use conn.Read, does the code need to modify?
3. Write an example that can cause the issue when we don't handle conn.Read properly
4. When a process creates a new process using the fork() operation, which
   of the following states is shared between the parent process and the child
   process?

   a. Stack

   b. Heap

   c. Shared memory segments

## Reference
- [Golang process and shared memory](https://stackoverflow.com/questions/41322376/golang-processes-and-shared-memory)
- [IPC: Shared memory concepts of C in Golang](https://medium.com/@glitchfix/ipc-shared-memory-concepts-of-c-in-golang-f539001226dc)
- [IPC: Shared memory concepts of C in Golang](https://medium.com/@caring_smitten_gerbil_914/pipes-as-conversations-understanding-ipc-through-named-and-anonymous-pipes-in-go-7d4aba28d28d)
- [Reading tcp socket](https://incoherency.co.uk/blog/stories/reading-tcp-sockets.html)