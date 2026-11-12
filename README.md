# ssh

SSH for Ferron programs, both sides, through the system `libssh` (0.12 or
newer). libssh does the protocol (key exchange, ciphers, authentication,
channels); this package is the safe layer over it:

- **`ssh.server`**: a server for terminal programs. `serve` listens and
  hands each authenticated connection with a shell to a handler as a
  `Session`.
- **`ssh.client`**: a client. `connect` checks the server's host key and
  authenticates; the `Client` then runs commands and drives interactive
  shells.

The two talk to each other and to OpenSSH (`scripts/interop` checks both).

## Use

Add it to the `anvil.toml` of a package as a git dependency pinned by tag:

```toml
[dependencies]
ssh = { git = "https://github.com/ferron-lang/ssh", tag = "v0.2.0" }
```

and import it as `ssh`. It needs `ferron` 0.10.0-alpha.3 or newer.
Versions stay `v0.x` until the API settles: the minor version marks
changes that may break callers, the patch version fixes; `CHANGELOG.md`
lists them.

Programs using the package link the system `libssh` (`[native] link` in
this package's `anvil.toml`), so `cc -lssh` must find version 0.12 or
newer:

| System | Install |
|---|---|
| Arch | `pacman -S libssh` |
| Debian 14 (forky), Ubuntu 26.10 and newer | `apt-get install libssh-dev` |
| macOS | `brew install libssh` |

Debian 13 and Ubuntu 26.04 and older ship libssh 0.11 or 0.10, which lack
functions this package binds (`ssh_pki_generate_key`, the
keyboard-interactive server callback); there, build libssh 0.12 from
[its sources](https://www.libssh.org/files/0.12/) with CMake, as this
package's CI does.

### A server

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
    password: None,
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
  whatever name the client typed and proves nothing. With
  `password: Some(check)`, `check(user, password)` decides who comes in by
  password, as `Password(user)`; a connection gets three attempts. The
  password reaches `check` as a `std.crypto.Secret`, whose bytes come out
  only through `expose_bytes()`.
- **The terminal.** `session.term()` and `session.size()` come from the
  client's `pty-req`, `session.variable("COLORTERM")` from its `SendEnv`.
  `poll` returns the bytes it typed (feed them to a `std.term.Decoder`),
  its window changes, and `Closed` at the end.
- **What is refused.** `exec`, subsystems (`sftp`), port forwarding, agent
  and X11 forwarding, and a second channel.
- **The host key** is an Ed25519 key in OpenSSH format, written with mode
  0600 on the first run.

### A client

```ferron
use ssh.client.{AcceptNew, Agent, ClientConfig, ExitStatus, KeyFile, Password, connect}
use ssh.error.SshError
use std.crypto.Secret
use std.env.Env
use std.fs.Path
use std.time.Duration

fn main(env: Env) -> Result[Unit, SshError] {
  let config = ClientConfig {
    host: "example.org",
    port: 22,
    user: "ann",
    timeout: Duration.seconds(10),
    host_key: AcceptNew(Path.from("known_hosts")),
    auth: [
      KeyFile(Path.from("id_ed25519"), None),
      Agent,
      Password(Secret.from_bytes(env.vars.get("ANN_PASSWORD").unwrap_or("").utf8_bytes())),
    ],
  }
  let client = connect(env.net, env.fs, env.clock, config)?
  let output = client.run("uname -a", Duration.seconds(30), 65536)?
  if output.status == ExitStatus.Code(0) {
    env.console.print(String.from_utf8_lossy(output.stdout))
  }
  return Ok(Unit)
}
```

`connect` reaches the server, checks its host key, and authenticates; the
`Client` is a resource and disconnects when its scope ends. A failure is
an `SshError` whose `kind` says which step failed: `Config` (nothing was
sent), `Connect`, `Timeout`, `HostKeyUnknown`, `HostKeyChanged`,
`KnownHosts`, `Authentication`, and later `Channel`, `TooLarge`, and
`Closed`. Every error is one `SshError { kind, message, cause }`; `cause`
holds the `ActorError` behind an `Actor` failure, which `source()` and
`AnyError.downcast` find. Compare errors by `kind`.

- **`client.run(command, timeout, max_output)`** runs a command in the
  user's shell, with nothing on its standard input, and returns a
  `CommandOutput`: `stdout` and `stderr` as `Bytes`, and `status`,
  an `ExitStatus`: `Code(code)`, `Signal(name)` for a command a signal
  killed, or `Unknown` when the server did not say. Both limits are
  required: a command that runs longer fails with `Timeout`, and one that
  writes more than `max_output` bytes to the two streams together fails
  with `TooLarge`, so a server cannot hold the caller or fill its memory.
- **`client.shell(pty, body)`** starts the user's shell on a terminal
  (`Pty { term, columns, rows }`) and lends it to `body` as a `Shell`;
  the shell's channel closes when `body` returns, and `shell` returns its
  result:

  ```ferron
  use ssh.client.{Client, Ended, ExitStatus, Pty, Shell, Stderr, Stdout, Unknown}
  use ssh.error.SshError
  use std.io.Console
  use std.time.Duration

  fn list_files(console: Console, client: Client) -> Result[ExitStatus, SshError] {
    let pty = Pty { term: "xterm-256color", columns: 80, rows: 24 }
    return client.shell(pty, fn(shell: mut Shell) -> Result[ExitStatus, SshError] {
      shell.write("ls; exit\n".utf8_bytes())?
      shell.resize(120, 40)?
      for _ in 0..600 {
        for event in shell.poll(Duration.milliseconds(100)) {
          match event {
            Stdout(bytes) => console.print(String.from_utf8_lossy(bytes))
            Stderr(_) => {
            }
            Ended(status) => return Ok(status)
          }
        }
      }
      return Ok(Unknown)
    })
  }
  ```

  `write` sends typed bytes, `poll(timeout)` returns what arrived
  (`Stdout`, `Stderr`, and `Ended(status)` once, at the end), and `resize`
  reports a window change. It is the other side of `ssh.session.Session`.
- **`client.host_key()`** is the server's fingerprint that the policy
  accepted, `SHA256:<base64>`.

## Security

- **The server's host key is always checked.** `ClientConfig.host_key` is
  one of three policies, and none accepts any key:
  `Strict(known_hosts)` wants the key already in that OpenSSH
  `known_hosts` file and fails with `HostKeyUnknown` otherwise;
  `AcceptNew(known_hosts)` records the key of a server it sees for the
  first time (creating the file) and from then on wants that key;
  `Fingerprint("SHA256:...")` pins one key and reads no file. With every
  policy a server whose key is not the recorded or pinned one fails with
  `HostKeyChanged`, before any credential is sent, and the file is left
  as it was.
- **Only what the config names is read.** The client reads no
  `~/.ssh/config` or `/etc/ssh/ssh_config`, no default identities, and no
  `known_hosts` but the policy's file. `Agent` uses the agent of the
  `SSH_AUTH_SOCK` environment variable.
- **Authentication** tries `ClientConfig.auth` in order: `KeyFile(path,
  passphrase)`, `Agent`, `Password(secret)`. When none lets the client
  in, the `Authentication` error says what each attempt came to and which
  methods the server accepts. An encrypted key without its passphrase
  fails; nothing ever prompts on the terminal.
- **Secrets.** Passwords and passphrases are `std.crypto.Secret` values on
  both sides: the client's `Auth.Password` and `Auth.KeyFile` passphrase,
  and the password the server hands to its `PasswordCheck`. A secret
  displays as `<secret>` (printing a config shows none, and no error
  message holds one), is affine (an `Auth` and the `ClientConfig` holding
  it move, they are not copied), and is zeroed when it is released. Its
  bytes leave it only through `expose_bytes()`, which this package calls
  where libssh needs the text and does not keep the result. libssh keeps
  its own copies while it uses them, which this package cannot wipe.
- **Limits.** Every client wait ends: `ClientConfig.timeout` bounds the
  connection and each answer of the server, `run` takes its own timeout
  and output cap, and `Shell.poll` returns about 1 MiB at most. The
  server bounds logins (`login_timeout`), connections
  (`max_connections`), password attempts, the terminal type, the window,
  and the environment a client may send.
- **Threads.** libssh's calls block, and it keeps a session to one
  thread. On the server each connection is an actor pinned to a thread of
  its own, and the handler runs there: a slow client stalls nobody else.
  The handler may capture only sendable values (capabilities, `Pid`s,
  channel ends); the `Session` is lent to it and stays on that thread.
  Call the client from `main` or from an actor spawned with
  `scope.spawn_pinned`; a `Client` is not sendable and stays where
  `connect` made it.

## Algorithms

libssh negotiates the transport, and this package keeps libssh 0.12's
defaults on both sides. Against OpenSSH 10 the server offers:

| | |
|---|---|
| Key exchange | `mlkem768x25519-sha256`, `mlkem768nistp256-sha256`, `mlkem1024nistp384-sha384`, `sntrup761x25519-sha512`, `curve25519-sha256`, `ecdh-sha2-nistp256/384/521`, `diffie-hellman-group18/16/14` and `group-exchange-sha256`, and `kex-strict-s-v00@openssh.com` (strict key exchange against Terrapin, CVE-2023-48795) |
| Host key | `ssh-ed25519` only: the server makes and loads one Ed25519 key |
| Ciphers | `chacha20-poly1305@openssh.com`, `aes256-gcm@openssh.com`, `aes128-gcm@openssh.com`, `aes256/192/128-ctr` |
| MACs (for the CTR ciphers) | `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-512-etm@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512` |
| Compression | `none`, `zlib@openssh.com` (after authentication only) |

Nothing with SHA-1, no CBC ciphers, no DSA or RSA host keys on the
server. The client accepts the host key types libssh 0.12 accepts
(Ed25519, ECDSA, RSA with SHA-2).

## Limits

- **libssh 0.12 or newer**, on Linux and macOS; Windows is untested.
- **One OS thread per connection** on the server, and one per `Client`:
  libssh's calls block, and Ferron has no pool for blocking C calls yet.
  Fine for tens or a few hundred connections; set `max_connections` to
  what the machine can hold.
- **The server serves terminal programs only**: one `session` channel
  with a shell. It refuses `exec`, subsystems (`sftp`), port forwarding,
  agent and X11 forwarding, and a second channel.
- **The client** runs commands and shells. It has no SFTP, no port
  forwarding, no `keyboard-interactive` login, and no host key types
  beyond libssh's.
- **Terminal input** arrives as raw bytes (`Input.Data`): decode it with
  `std.term.Decoder`, which also handles a lone Esc
  (`Decoder.ESCAPE_TIMEOUT`, `expire`).

## Develop

Needs the libssh headers (`libssh/libssh.h` in `/usr/include`) and
`clang` (for `anvil ffi gen`).

```sh
anvil check
anvil test          # every module's tests; `anvil test pkg.client` filters
anvil fmt
anvil ffi gen --check
scripts/interop     # the client against OpenSSH and against ssh.server
```

`scripts/interop` builds `examples/probe`, a command-line program over
both sides of the package, and drives it: against a private, unprivileged
OpenSSH `sshd` on 127.0.0.1 port 2300 (the three host key policies, a
changed host key, key, agent, and password logins, a command's output,
status, cap, and timeout, a shell with a resize) and against the probe's
own `ssh.server` on port 2301 (a key and a password login, bytes both
ways, a resize, a clean close). It prints `ok` or `FAIL` per check and
exits with status 1 when one failed. It needs `sshd`, `ssh-keygen`, and
`ssh-agent` on the machine, keeps its keys and `known_hosts` in a
temporary directory that it removes, stops what it started, and never
touches `~/.ssh`. `SSH_INTEROP_PORT` moves the two ports, and `ANVIL`
names the `anvil` to build the probe with.

## C bindings

`anvil ffi gen` writes the `extern "C"` modules in `src/sys/` from the
libssh headers. `ffi.toml` pins the binding to the functions the safe
layer calls (`ignore = "."` plus a `require` list), so the output is the
same for the libssh 0.12.0 and 0.12.2 headers; using another function
means adding it to `require`. `[nullable]` lets the key file functions
take a NULL passphrase. `src/sys/hand.fer` declares libc's `umask`, the
one call outside libssh.

## CI

`.github/workflows/ci.yml` builds the compiler from `ferron-lang/ferron` at
`FERRON_REF`, a release tag or a commit, installs `clang` and OpenSSH,
builds libssh at `LIBSSH_VERSION` from its sources (Ubuntu's runners ship
0.10 and 0.11), and runs `anvil check`, `anvil test` (checking that the
runtime reports `live_handles=0` afterwards), `anvil fmt --check`, the
same two checks of `examples/probe`, `scripts/interop`, and
`anvil ffi gen --check`. `ferron-lang/ferron` is private, so the workflow
checks it out with the repository secret `FERRON_REPO_TOKEN`: a fine-grained
personal access token with read access to the contents of
`ferron-lang/ferron`.

## License

MIT; see `LICENSE`.
