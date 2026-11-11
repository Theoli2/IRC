# ft_irc — IRC server in C++98 *(work in progress)*

An IRC server written from scratch in C++98 as part of the 42 curriculum.
It handles several clients at once in a single thread, using **non-blocking sockets** and an **`epoll` event loop**.

> **Status:** the networking core and client registration work. Channels and message routing between users are still on the roadmap (see below).

## Build & run

```bash
make        # builds ./IRC
./IRC       # listens on port 6667
```

Then connect with any IRC client (irssi, WeeChat, HexChat…) to `localhost:6667`, or test by hand with netcat:

```bash
nc localhost 6667
NICK alice
USER alice 0 * :Alice
```

## How it works

```
socket() → fcntl(O_NONBLOCK) → bind() → listen()
                     │
                epoll_create1()
                     │
        ┌──── epoll_wait() ◄────────────┐
        │                               │
 listening socket ready?          client socket ready?
        │                               │
   accept() new client             read() → parse the command
   set non-blocking                → send the reply
   add it to epoll ────────────────────►┘
```

- **One thread, many clients**: the server never blocks on a single client. `epoll_wait` reports which sockets are ready, and the loop only touches those.
- **Non-blocking I/O**: the listening socket and every client socket are switched to `O_NONBLOCK` with `fcntl`.
- **Per-client state**: nicknames and usernames are kept in `std::map<int, std::string>` indexed by the client's file descriptor, and cleaned up when the client disconnects.
- **Error handling**: when a system call fails during setup, the file descriptors already opened are closed and a `std::runtime_error` is thrown, then caught in `main`, which exits with an error message.

## Supported commands

| Command | Behaviour |
|---|---|
| `CAP LS` / `CAP END` | Capability negotiation sent by modern clients on connect |
| `NICK` + `USER` | Registration; the server answers with `001 RPL_WELCOME` once both are received |
| `PING` | Answers with `PONG` (keeps the client connected) |
| `MODE` | Returns the user mode (`221 +i`) |
| `WHOIS` | Looks up a connected nickname (`311` / `318`, or `401 No such nick`) |
| `PRIVMSG` | Parsed; CTCP `PING` is answered |
| `QUIT` | Closes the connection and frees the client's state |
| anything else | `421 Unknown command` |

## Roadmap

- [ ] Take the port and a connection password as arguments (`./ircserv <port> <password>`, `PASS`)
- [ ] Buffer partial reads so commands split across several packets (or several commands in one packet) are handled correctly
- [ ] Deliver `PRIVMSG` to the target user or channel
- [ ] Channels: `JOIN`, `PART`, `TOPIC`, `KICK`, `INVITE`, channel `MODE` (`i`, `t`, `k`, `o`, `l`)

## Project structure

```
.
├── Makefile
├── includes/irc.h     # shared headers and prototypes
└── srcs/
    ├── main.cpp       # command handling and entry point
    ├── socket.cpp     # socket creation, non-blocking mode, bind, listen
    └── epoll.cpp      # epoll instance and the event loop
```

## Tech

C++98 · POSIX sockets · `epoll` · Linux
