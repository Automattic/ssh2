# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project overview

`ssh2` is an SSH2 client and server implementation for Node.js, written in
JavaScript (CommonJS) with an optional native crypto addon. It is published to
npm as `ssh2`. The only runtime dependencies are `asn1` and `bcrypt-pbkdf`;
`cpu-features` and `nan` are optional.

- Supported Node.js: **>= 16** (`engines` in `package.json`). CI tests 16.x
  through 26.x on Linux, macOS and Windows. Don't use language or API features
  newer than Node 16 without a fallback.
- User-facing API docs live in `README.md` (client, server, agents, keygen) and
  `SFTP.md` (SFTP methods). There is no generated documentation.

## Layout

```
lib/
  index.js            Public exports (Client, Server, agents, HTTP(S) agents, utils)
  client.js           Client class (connect, auth, channels, forwarding, sftp())
  server.js           Server + IncomingClient + auth contexts
  Channel.js          Duplex stream for SSH channels
  agent.js            ssh-agent protocol and agent implementations
  http-agents.js      http(s).Agent that tunnels over SSH
  keygen.js           generateKeyPair / generateKeyPairSync
  utils.js            Shared helpers for client/server
  protocol/
    Protocol.js       Transport layer: packet parsing, sending, rekeying
    kex.js            Key exchange (KEXINIT, DH/ECDH/curve25519, NEWKEYS)
    handlers.js       Builds the message-type -> handler table
    handlers.misc.js  Handlers for non-KEX SSH messages
    crypto.js         Cipher/decipher factories (binding or pure JS)
    crypto/           Optional native addon (C++/nan) + WASM poly1305
    keyParser.js      Parses OpenSSH, PPK, PEM and public keys
    SFTP.js           SFTP client and server (v3)
    constants.js      Message IDs, algorithm lists, defaults
    zlib.js           zlib compression for the transport
    utils.js          Low-level buffer parsing/writing helpers
test/                 One standalone script per test-*.js
examples/             Example servers
install.js            npm install hook: tries to build the crypto addon
```

## Setup, build, test, lint

```sh
npm install        # also runs install.js -> node-gyp builds lib/protocol/crypto
npm run rebuild    # rebuild the native crypto addon only
npm test           # node test/test.js: runs every test/test-*.js in its own process
npm run lint       # eslint over .eslintrc.js, examples, lib, test
npm run lint:fix
```

- If the addon fails to build, `install.js` prints a message and exits 0, so the
  library falls back to pure-JS crypto. Set `CI_CHECK_FAIL=ssh2` to make a build
  failure fatal (CI does this).
- CI also sets `OPENSSL_CONF=/dev/null`. Do the same locally if your system
  OpenSSL config breaks tests.
- To run a single test: `node test/test-sftp.js`. Tests exit non-zero on failure;
  there is no test framework.
- Some tests need external tools and skip themselves when those are missing
  (for example `test-integration-openssh.js` needs the `ssh` client and skips
  on Windows).

## Code style

Linting uses `@mscdex/eslint-config` (`.eslintrc.js`).
`lib/protocol/crypto/poly1305.js` is generated code and is ignored by ESLint;
don't edit it by hand. Match the existing code:

- `'use strict';` at the top of every file; CommonJS `require` /
  `module.exports`.
- 2-space indent, single quotes, semicolons, lines at most 80 columns.
- Continuation lines line up with the opening bracket or operand (see
  existing multi-line conditions and calls). Single-statement `if` bodies
  usually go without braces.
- Destructure imports, sorted alphabetically when there are many:
  `const { bufferSlice, writeUInt32BE } = require('./utils.js');`
- Classes usually extend `EventEmitter`. Private state uses a leading
  underscore (`this._sock`). Module-level helpers that take `self` as their first
  argument are common instead of extra methods.
- Protocol code is performance-sensitive. Reuse the shared parsers
  (`bufferParser`, `sigSSHToASN1`, `readUInt32BE`, `writeUInt32BE` from
  `lib/protocol/utils.js`), avoid needless allocations in per-packet paths, and
  keep the RFC wire-format comment blocks above handlers (see
  `handlers.misc.js`).
- Include the file extension in `require()` paths (`'./utils.js'`).

## Tests

- Each `test/test-*.js` is a plain Node script using `assert` and the helpers
  in `test/common.js`:
  - `mustCall(fn, n)` / `mustCallAtLeast(fn, n)` / `mustNotCall()` check
    callback counts when the process exits.
  - `setup(title, { client, server, debug, ... })` and
    `setupSimple(debug, title)` start a connected client/server pair over a
    local socket.
  - `fixture(name)` / `fixtureKey(name)` load files from `test/fixtures/`.
- Tests are grouped in `{ ... }` blocks with a descriptive title. Add new cases
  the same way, in the test file for that area.
- Key parser tests are driven by fixtures. For each key in
  `test/fixtures/keyParser/` there is a `<name>.result` file holding the expected
  parsed JSON (or an error message string). Adding a key format means adding
  both files.
- Crypto changes must pass `test/test-protocol-crypto.js`, which round-trips
  every cipher/MAC combination between the native binding and the pure-JS
  implementation (`forceNative`).

## Crypto

`lib/protocol/crypto.js` has two implementations of each cipher:
`*Binding` (native addon from `lib/protocol/crypto/src/binding.cc`) and
`*Native` (Node `crypto` + WASM Poly1305). Factories pick the binding when it
loaded, unless `forceNative` is set. Changes to the cipher logic must keep both
implementations in sync and produce identical output.

`bindingAvailable` also affects the default cipher order
(`lib/protocol/constants.js`), together with `cpu-features`.

Native code builds against Node's bundled OpenSSL via `nan`. It is compiled
with `OPENSSL_API_COMPAT=0x10100000L` for OpenSSL 3.

## Commits and PRs

- Commit subjects start with the area(s) touched, then a short imperative
  summary, e.g. `crypto: use nan's ObjectWrap`,
  `ci,package,readme,test: drop support for node < v16`. Common prefixes: `lib`,
  `client`, `server`, `protocol/kex`, `keyParser`, `SFTP`, `crypto`, `test`,
  `readme`, `ci`, `package`.
- Update `README.md` / `SFTP.md` when public API behavior or options change.
- Keep the Node 16 floor unless the change explicitly drops it (as was done in
  `ci,package,readme,test: drop support for node < v16`).
- Don't commit build output (`lib/protocol/crypto/build/`) or `node_modules`.
