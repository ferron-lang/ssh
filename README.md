# ssh

An SSH server for terminal programs written in Ferron, through the system
`libssh` (0.12 or newer). libssh does the protocol (key exchange, ciphers,
authentication, the channel); this package hands each authenticated
connection with a shell to a handler as a `Session`.

It has the server side only: it accepts connections and does not make them.

## Use

```toml
[dependencies]
ssh = { path = "../ssh" }
```

```ferron
use ssh.error.SshError
use ssh.server.{Config, serve}
use ssh.session.{Closed, Data, Resize, Session}
use std.env.Env
use std.fs.Path
use std.time.Duration

fn main(env: Env) -> Result[Unit, SshError] {
  let config = Config {
    address: "127.0.0.1",
    port: 2222,
    host_key: Path.from("host_key"),
    max_connections: 64,
    allow_guests: true,
    login_timeout: Duration.seconds(60),
  }
  return serve(env.actors, env.clock, env.fs, config, fn(session: mut Session) -> Unit {
    let _ = session.write("hello {session.user()}\r\n".utf8_bytes())
    var open = true
    while open {
      for input in session.poll(Duration.milliseconds(500)) {
        match input {
          Data(bytes) => {
            let _ = session.write(bytes)
          }
          Resize(_, _) => {
          }
          Closed => open = false
        }
      }
    }
  })
}
```

- **Who the client is.** Any public key is accepted, and the client must
  prove it holds the private key: `session.identity()` is then
  `Key(algorithm, fingerprint)`, the same fingerprint `ssh-keygen -l`
  prints, and it names the same client on every connection. With
  `allow_guests`, a client without a key comes in through
  `keyboard-interactive` with no prompt as `Guest`. `session.user()` is
  whatever name the client typed and proves nothing.
- **The terminal.** `session.term()` and `session.size()` come from the
  client's `pty-req`, `session.variable("COLORTERM")` from its `SendEnv`.
  `poll` returns the bytes it typed (feed them to a `std.term.Decoder`),
  its window changes, and `Closed` at the end.
- **Threads.** libssh's calls block, so each connection is an actor pinned
  to a thread of its own, and the handler runs there: a slow client stalls
  nobody else. The handler may capture only sendable values (capabilities,
  `Pid`s, channel ends); the `Session` is lent to it and stays on that
  thread.
- **What is refused.** `exec`, subsystems (`sftp`), port forwarding, agent
  and X11 forwarding, and a second channel.
- **The host key** is an Ed25519 key in OpenSSH format, written with mode
  0600 on the first run.

## Develop

```sh
anvil check
anvil test
anvil fmt
anvil ffi gen --check
```

`anvil ffi gen` writes the `extern "C"` modules in `src/sys/` from the
libssh headers; `ffi.toml` lists the functions the package binds, and
`src/sys/hand.fer` holds the two declared by hand.

## License

MIT; see `LICENSE`.
