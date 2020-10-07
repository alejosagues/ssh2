# WARNING: This documentation is for an upcoming, unreleased version of `ssh2`. You are probably looking for documentation for `ssh2` v0.8.x [here](https://github.com/mscdex/ssh2/blob/v0.8.x/README.md)

# Description

SSH2 client and server modules written in pure JavaScript for [node.js](http://nodejs.org/).

Development/testing is done against OpenSSH (8.0 currently).

[![Build Status](https://travis-ci.org/mscdex/ssh2.svg?branch=master)](https://travis-ci.org/mscdex/ssh2)

# Table of Contents

* [Requirements](#requirements)
* [Installation](#installation)
* [Client Examples](#client-examples)
  * [Execute 'uptime' on a server](#execute-uptime-on-a-server)
  * [Start an interactive shell session](#start-an-interactive-shell-session)
  * [Send a raw HTTP request to port 80 on the server](#send-a-raw-http-request-to-port-80-on-the-server)
  * [Forward local connections to port 8000 on the server to us](#forward-local-connections-to-port-8000-on-the-server-to-us)
  * [Get a directory listing via SFTP](#get-a-directory-listing-via-sftp)
  * [Connection hopping](#connection-hopping)
  * [Forward remote X11 connections](#forward-remote-x11-connections)
  * [Dynamic (1:1) port forwarding using a SOCKSv5 proxy (using `socksv5`)](#dynamic-11-port-forwarding-using-a-socksv5-proxy-using-socksv5)
  * [Make HTTP(S) connections easily using a custom http(s).Agent](#make-https-connections-easily-using-a-custom-httpsagent)
  * [Invoke an arbitrary subsystem (e.g. netconf)](#invoke-an-arbitrary-subsystem)
* [Server Examples](#server-examples)
  * [Password and public key authentication and non-interactive (exec) command execution](#password-and-public-key-authentication-and-non-interactive-exec-command-execution)
  * [SFTP-only server](#sftp-only-server)
* [API](#api)
  * [Client](#client)
      * [Client events](#client-events)
      * [Client methods](#client-methods)
  * [Server](#server)
      * [Server events](#server-events)
      * [Server methods](#server-methods)
      * [Connection events](#connection-events)
      * [Connection methods](#connection-methods)
      * [Session events](#session-events)
  * [Channel](#channel)
  * [Pseudo-TTY settings](#pseudo-tty-settings)
  * [Terminal modes](#terminal-modes)
  * [HTTPAgent](#httpagent)
      * [HTTPAgent methods](#httpagent-methods)
  * [HTTPSAgent](#httpsagent)
      * [HTTPSAgent methods](#httpsagent-methods)

## Requirements

* [node.js](http://nodejs.org/) -- v10.16.0 or newer
  * node v12.0.0 or newer for Ed25519 key support

## Installation

    npm install ssh2

## Client Examples

### Execute 'uptime' on a server

```js
const { readFileSync } = require('fs');

const { Client } = require('ssh2');

const conn = new Client();
conn.on('ready', () => {
  console.log('Client :: ready');
  conn.exec('uptime', (err, stream) => {
    if (err) throw err;
    stream.on('close', (code, signal) => {
      console.log('Stream :: close :: code: ' + code + ', signal: ' + signal);
      conn.end();
    }).on('data', (data) => {
      console.log('STDOUT: ' + data);
    }).stderr.on('data', (data) => {
      console.log('STDERR: ' + data);
    });
  });
}).connect({
  host: '192.168.100.100',
  port: 22,
  username: 'frylock',
  privateKey: readFileSync('/path/to/my/key')
});

// example output:
// Client :: ready
// STDOUT:  17:41:15 up 22 days, 18:09,  1 user,  load average: 0.00, 0.01, 0.05
//
// Stream :: exit :: code: 0, signal: undefined
// Stream :: close
```

### Start an interactive shell session

```js
const { readFileSync } = require('fs');

const { Client } = require('ssh2');

const conn = new Client();
conn.on('ready', () => {
  console.log('Client :: ready');
  conn.shell((err, stream) => {
    if (err) throw err;
    stream.on('close', () => {
      console.log('Stream :: close');
      conn.end();
    }).on('data', (data) => {
      console.log('OUTPUT: ' + data);
    });
    stream.end('ls -l\nexit\n');
  });
}).connect({
  host: '192.168.100.100',
  port: 22,
  username: 'frylock',
  privateKey: readFileSync('/path/to/my/key')
});

// example output:
// Client :: ready
// STDOUT: Last login: Sun Jun 15 09:37:21 2014 from 192.168.100.100
//
// STDOUT: ls -l
// exit
//
// STDOUT: frylock@athf:~$ ls -l
//
// STDOUT: total 8
//
// STDOUT: drwxr-xr-x 2 frylock frylock 4096 Nov 18  2012 mydir
//
// STDOUT: -rw-r--r-- 1 frylock frylock   25 Apr 11  2013 test.txt
//
// STDOUT: frylock@athf:~$ exit
//
// STDOUT: logout
//
// Stream :: close
```

### Send a raw HTTP request to port 80 on the server

```js
const { Client } = require('ssh2');

const conn = new Client();
conn.on('ready', () => {
  console.log('Client :: ready');
  conn.forwardOut('192.168.100.102', 8000, '127.0.0.1', 80, (err, stream) => {
    if (err) throw err;
    stream.on('close', () => {
      console.log('TCP :: CLOSED');
      conn.end();
    }).on('data', (data) => {
      console.log('TCP :: DATA: ' + data);
    }).end([
      'HEAD / HTTP/1.1',
      'User-Agent: curl/7.27.0',
      'Host: 127.0.0.1',
      'Accept: */*',
      'Connection: close',
      '',
      ''
    ].join('\r\n'));
  });
}).connect({
  host: '192.168.100.100',
  port: 22,
  username: 'frylock',
  password: 'nodejsrules'
});

// example output:
// Client :: ready
// TCP :: DATA: HTTP/1.1 200 OK
// Date: Thu, 15 Nov 2012 13:52:58 GMT
// Server: Apache/2.2.22 (Ubuntu)
// X-Powered-By: PHP/5.4.6-1ubuntu1
// Last-Modified: Thu, 01 Jan 1970 00:00:00 GMT
// Content-Encoding: gzip
// Vary: Accept-Encoding
// Connection: close
// Content-Type: text/html; charset=UTF-8
//
//
// TCP :: CLOSED
```

### Forward local connections to port 8000 on the server to us

```js
const { Client } = require('ssh2');

const conn = new Client();
conn.on('ready', () => {
  console.log('Client :: ready');
  conn.forwardIn('127.0.0.1', 8000, (err) => {
    if (err) throw err;
    console.log('Listening for connections on server on port 8000!');
  });
}).on('tcp connection', (info, accept, reject) => {
  console.log('TCP :: INCOMING CONNECTION:');
  console.dir(info);
  accept().on('close', () => {
    console.log('TCP :: CLOSED');
  }).on('data', (data) => {
    console.log('TCP :: DATA: ' + data);
  }).end([
    'HTTP/1.1 404 Not Found',
    'Date: Thu, 15 Nov 2012 02:07:58 GMT',
    'Server: ForwardedConnection',
    'Content-Length: 0',
    'Connection: close',
    '',
    ''
  ].join('\r\n'));
}).connect({
  host: '192.168.100.100',
  port: 22,
  username: 'frylock',
  password: 'nodejsrules'
});

// example output:
// Client :: ready
// Listening for connections on server on port 8000!
//  (.... then from another terminal on the server: `curl -I http://127.0.0.1:8000`)
// TCP :: INCOMING CONNECTION: { destIP: '127.0.0.1',
//  destPort: 8000,
//  srcIP: '127.0.0.1',
//  srcPort: 41969 }
// TCP DATA: HEAD / HTTP/1.1
// User-Agent: curl/7.27.0
// Host: 127.0.0.1:8000
// Accept: */*
//
//
// TCP :: CLOSED
```

### Get a directory listing via SFTP

```js
const { Client } = require('ssh2');

const conn = new Client();
conn.on('ready', () => {
  console.log('Client :: ready');
  conn.sftp((err, sftp) => {
    if (err) throw err;
    sftp.readdir('foo', (err, list) => {
      if (err) throw err;
      console.dir(list);
      conn.end();
    });
  });
}).connect({
  host: '192.168.100.100',
  port: 22,
  username: 'frylock',
  password: 'nodejsrules'
});

// example output:
// Client :: ready
// [ { filename: 'test.txt',
//     longname: '-rw-r--r--    1 frylock   frylock         12 Nov 18 11:05 test.txt',
//     attrs:
//      { size: 12,
//        uid: 1000,
//        gid: 1000,
//        mode: 33188,
//        atime: 1353254750,
//        mtime: 1353254744 } },
//   { filename: 'mydir',
//     longname: 'drwxr-xr-x    2 frylock   frylock       4096 Nov 18 15:03 mydir',
//     attrs:
//      { size: 1048576,
//        uid: 1000,
//        gid: 1000,
//        mode: 16877,
//        atime: 1353269007,
//        mtime: 1353269007 } } ]
```

### Connection hopping

```js
const { Client } = require('ssh2');

const conn1 = new Client();
const conn2 = new Client();

// Checks uptime on 10.1.1.40 via 192.168.1.1

conn1.on('ready', () => {
  console.log('FIRST :: connection ready');
  // Alternatively, you could use something like netcat or socat with exec()
  // instead of forwardOut(), depending on what the server allows
  conn1.forwardOut('127.0.0.1', 12345, '10.1.1.40', 22, (err, stream) => {
    if (err) {
      console.log('FIRST :: forwardOut error: ' + err);
      return conn1.end();
    }
    conn2.connect({
      sock: stream,
      username: 'user2',
      password: 'password2',
    });
  });
}).connect({
  host: '192.168.1.1',
  username: 'user1',
  password: 'password1',
});

conn2.on('ready', () => {
  // This connection is the one to 10.1.1.40

  console.log('SECOND :: connection ready');
  conn2.exec('uptime', (err, stream) => {
    if (err) {
      console.log('SECOND :: exec error: ' + err);
      return conn1.end();
    }
    stream.on('close', () => {
      conn1.end(); // close parent (and this) connection
    }).on('data', (data) => {
      console.log(data.toString());
    });
  });
});
```

### Forward remote X11 connections

```js
const { Socket } = require('net');

const { Client } = require('ssh2');

const conn = new Client();

conn.on('x11', (info, accept, reject) => {
  const xserversock = new net.Socket();
  xserversock.on('connect', () => {
    const xclientsock = accept();
    xclientsock.pipe(xserversock).pipe(xclientsock);
  });
  // connects to localhost:0.0
  xserversock.connect(6000, 'localhost');
});

conn.on('ready', () => {
  conn.exec('xeyes', { x11: true }, (err, stream) => {
    if (err) throw err;
    let code = 0;
    stream.on('close', () => {
      if (code !== 0)
        console.log('Do you have X11 forwarding enabled on your SSH server?');
      conn.end();
    }).on('exit', (exitcode) => {
      code = exitcode;
    });
  });
}).connect({
  host: '192.168.1.1',
  username: 'foo',
  password: 'bar'
});
```

### Dynamic (1:1) port forwarding using a SOCKSv5 proxy (using [socksv5](https://github.com/mscdex/socksv5))

```js
const socks = require('socksv5');
const { Client } = require('ssh2');

const sshConfig = {
  host: '192.168.100.1',
  port: 22,
  username: 'nodejs',
  password: 'rules'
};

socks.createServer((info, accept, deny) => {
  // NOTE: you could just use one ssh2 client connection for all forwards, but
  // you could run into server-imposed limits if you have too many forwards open
  // at any given time
  const conn = new Client();
  conn.on('ready', () => {
    conn.forwardOut(info.srcAddr,
                    info.srcPort,
                    info.dstAddr,
                    info.dstPort,
                    (err, stream) => {
      if (err) {
        conn.end();
        return deny();
      }

      const clientSocket = accept(true);
      if (clientSocket) {
        stream.pipe(clientSocket).pipe(stream).on('close', () => {
          conn.end();
        });
      } else {
        conn.end();
      }
    });
  }).on('error', (err) => {
    deny();
  }).connect(sshConfig);
}).listen(1080, 'localhost', () => {
  console.log('SOCKSv5 proxy server started on port 1080');
}).useAuth(socks.auth.None());

// test with cURL:
//   curl -i --socks5 localhost:1080 google.com
```

### Make HTTP(S) connections easily using a custom http(s).Agent

```js
const http = require('http');

const { Client, HTTPAgent, HTTPSAgent } = require('ssh2');

const sshConfig = {
  host: '192.168.100.1',
  port: 22,
  username: 'nodejs',
  password: 'rules'
};

// Use `HTTPSAgent` instead for an HTTPS request
const agent = new HTTPAgent(sshConfig);
http.get({
  host: '192.168.200.1',
  agent,
  headers: { Connection: 'close' }
}, (res) => {
  console.log(res.statusCode);
  console.dir(res.headers);
  res.resume();
});
```


### Invoke an arbitrary subsystem

```js
const { Client } = require('ssh2');

const xmlhello = `
  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <capabilities>
      <capability>urn:ietf:params:netconf:base:1.0</capability>
    </capabilities>
  </hello>]]>]]>`;

const conn = new Client();

conn.on('ready', () => {
  console.log('Client :: ready');
  conn.subsys('netconf', (err, stream) => {
    if (err) throw err;
    stream.on('data', (data) => {
      console.log(data);
    }).write(xmlhello);
  });
}).connect({
  host: '1.2.3.4',
  port: 22,
  username: 'blargh',
  password: 'honk'
});
```

## Server Examples

### Password and public key authentication and non-interactive (exec) command execution

```js
const { timingSafeEqual } = require('crypto');
const { readFileSync } = require('fs');
const { inspect } = require('util');

const { parseKey, Server } = require('ssh2');

const allowedUser = Buffer.from('foo');
const allowedPassword = Buffer.from('bar');
const allowedPubKey = parseKey(readFileSync('foo.pub'));

function checkValue(input, allowed) {
  const autoReject = (input.length !== allowed.length);
  if (autoReject) {
    // Prevent leaking length information by always making a comparison with the
    // same input when lengths don't match what we expect ...
    allowed = input;
  }
  const isMatch = timingSafeEqual(input, allowed);
  return (!autoReject && isMatch);
}

new Server({
  hostKeys: [readFileSync('host.key')]
}, (client) => {
  console.log('Client connected!');

  client.on('authentication', (ctx) => {
    let allowed = true;
    if (!checkValue(Buffer.from(ctx.username), allowedUser))
      allowed = false;

    switch (ctx.method) {
      case 'password':
        if (!checkValue(Buffer.from(ctx.password), allowedPassword))
          return ctx.reject();
        break;
      case 'publickey':
        if (ctx.key.algo !== allowedPubKey.type
            || !checkValue(ctx.key.data, allowedPubKey.getPublicSSH())
            || (ctx.signature && allowedPubKey.verify(ctx.blob, ctx.signature) !== true)) {
          return ctx.reject();
        }
        break;
      default:
        return ctx.reject();
    }

    if (allowed)
      ctx.accept();
    else
      ctx.reject();
  }).on('ready', () => {
    console.log('Client authenticated!');

    client.on('session', (accept, reject) => {
      const session = accept();
      session.once('exec', (accept, reject, info) => {
        console.log('Client wants to execute: ' + inspect(info.command));
        const stream = accept();
        stream.stderr.write('Oh no, the dreaded errors!\n');
        stream.write('Just kidding about the errors!\n');
        stream.exit(0);
        stream.end();
      });
    });
  }).on('close', () => {
    console.log('Client disconnected');
  });
}).listen(0, '127.0.0.1', function() {
  console.log('Listening on port ' + this.address().port);
});
```

### SFTP-only server

```js
const { timingSafeEqual } = require('crypto');
const { readFileSync } = require('fs');
const { inspect } = require('util');

const {
  Server,
  sftp: {
    OPEN_MODE,
    STATUS_CODE,
  },
} = require('ssh2');

const allowedUser = Buffer.from('foo');
const allowedPassword = Buffer.from('bar');

function checkValue(input, allowed) {
  const autoReject = (input.length !== allowed.length);
  if (autoReject) {
    // Prevent leaking length information by always making a comparison with the
    // same input when lengths don't match what we expect ...
    allowed = input;
  }
  const isMatch = timingSafeEqual(input, allowed);
  return (!autoReject && isMatch);
}

// This simple SFTP server implements file uploading where the contents get
// ignored ...

new ssh2.Server({
  hostKeys: [readFileSync('host.key')]
}, (client) => {
  console.log('Client connected!');

  client.on('authentication', (ctx) => {
    let allowed = true;
    if (!checkValue(Buffer.from(ctx.username), allowedUser))
      allowed = false;

    switch (ctx.method) {
      case 'password':
        if (!checkValue(Buffer.from(ctx.password), allowedPassword))
          return ctx.reject();
        break;
      default:
        return ctx.reject();
    }

    if (allowed)
      ctx.accept();
    else
      ctx.reject();
  }).on('ready', () => {
    console.log('Client authenticated!');

    client.on('session', (accept, reject) => {
      const session = accept();
      session.on('sftp', (accept, reject) => {
        console.log('Client SFTP session');
        const openFiles = new Map();
        let handleCount = 0;
        const sftp = accept();
        sftp.on('OPEN', (reqid, filename, flags, attrs) => {
          // Only allow opening /tmp/foo.txt for writing
          if (filename !== '/tmp/foo.txt' || !(flags & OPEN_MODE.WRITE))
            return sftp.status(reqid, STATUS_CODE.FAILURE);

          // Create a fake handle to return to the client, this could easily
          // be a real file descriptor number for example if actually opening
          // a file on disk
          const handle = Buffer.alloc(4);
          openFiles.set(handleCount, true);
          handle.writeUInt32BE(handleCount++, 0);

          console.log('Opening file for write')
          sftp.handle(reqid, handle);
        }).on('WRITE', (reqid, handle, offset, data) => {
          if (handle.length !== 4
              || !openFiles.has(handle.readUInt32BE(0))) {
            return sftp.status(reqid, STATUS_CODE.FAILURE);
          }

          // Fake the write operation
          sftp.status(reqid, STATUS_CODE.OK);

          console.log('Write to file at offset ${offset}: ${inspect(data)}');
        }).on('CLOSE', (reqid, handle) => {
          let fnum;
          if (handle.length !== 4
              || !openFiles.has(fnum = handle.readUInt32BE(0))) {
            return sftp.status(reqid, STATUS_CODE.FAILURE);
          }

          console.log('Closing file');
          openFiles.delete(fnum);

          sftp.status(reqid, STATUS_CODE.OK);
        });
      });
    });
  }).on('close', () => {
    console.log('Client disconnected');
  });
}).listen(0, '127.0.0.1', function() {
  console.log('Listening on port ' + this.address().port);
});
```

You can find more examples in the `examples` directory of this repository.

## API

`require('ssh2').Client` returns the **_Client_** constructor.

`require('ssh2').Server` returns the **_Server_** constructor.

`require('ssh2').utils` returns an object containing some useful [utilities](#utilities).

`require('ssh2').HTTPAgent` returns an [`http.Agent`](https://nodejs.org/docs/latest/api/http.html#http_class_http_agent) constructor.

`require('ssh2').HTTPSAgent` returns an [`https.Agent`](https://nodejs.org/docs/latest/api/https.html#https_class_https_agent) constructor. Its API is the same as `HTTPAgent` except it's for HTTPS connections.

### Client

#### Client events

* **banner**(< _string_ >message, < _string_ >language) - A notice was sent by the server upon connection.

* **ready**() - Authentication was successful.

* **tcp connection**(< _object_ >details, < _function_ >accept, < _function_ >reject) - An incoming forwarded TCP connection is being requested. Calling `accept` accepts the connection and returns a `Channel` object. Calling `reject` rejects the connection and no further action is needed. `details` contains:

    * **srcIP** - _string_ - The originating IP of the connection.

    * **srcPort** - _integer_ - The originating port of the connection.

    * **destIP** - _string_ - The remote IP the connection was received on (given in earlier call to `forwardIn()`).

    * **destPort** - _integer_ - The remote port the connection was received on (given in earlier call to `forwardIn()`).

* **x11**(< _object_ >details, < _function_ >accept, < _function_ >reject) - An incoming X11 connection is being requested. Calling `accept` accepts the connection and returns a `Channel` object. Calling `reject` rejects the connection and no further action is needed. `details` contains:

    * **srcIP** - _string_ - The originating IP of the connection.

    * **srcPort** - _integer_ - The originating port of the connection.

* **keyboard-interactive**(< _string_ >name, < _string_ >instructions, < _string_ >instructionsLang, < _array_ >prompts, < _function_ >finish) - The server is asking for replies to the given `prompts` for keyboard-interactive user authentication. `name` is generally what you'd use as a window title (for GUI apps). `prompts` is an array of `{ prompt: 'Password: ', echo: false }` style objects (here `echo` indicates whether user input should be displayed on the screen). The answers for all prompts must be provided as an array of strings and passed to `finish` when you are ready to continue. Note: It's possible for the server to come back and ask more questions.

* **unix connection**(< _object_ >details, < _function_ >accept, < _function_ >reject) - An incoming forwarded UNIX socket connection is being requested. Calling `accept` accepts the connection and returns a `Channel` object. Calling `reject` rejects the connection and no further action is needed. `details` contains:

    * **socketPath** - _string_ - The originating UNIX socket path of the connection.

* **change password**(< _string_ >message, < _string_ >language, < _function_ >done) - If using password-based user authentication, the server has requested that the user's password be changed. Call `done` with the new password.

* **handshake**(< _object_ >negotiated) - Emitted when a handshake has completed (either initial or rekey). `negotiated` contains the negotiated details of the handshake and is of the form:

```js
    // In this particular case `mac` is empty because there is no separate MAC
    // because it's integrated into AES in GCM mode
    { kex: 'ecdh-sha2-nistp256',
      srvHostKey: 'rsa-sha2-512',
      cs: { // Client to server algorithms
        cipher: 'aes128-gcm',
        mac: '',
        compress: 'none',
        lang: ''
      },
      sc: { // Server to client algorithms
        cipher: 'aes128-gcm',
        mac: '',
        compress: 'none',
        lang: ''
      }
    }
```   

* **rekey**() - Emitted when a rekeying operation has completed (either client or server-initiated).

* **error**(< _Error_ >err) - An error occurred. A 'level' property indicates 'client-socket' for socket-level errors and 'client-ssh' for SSH disconnection messages. In the case of 'client-ssh' messages, there may be a 'description' property that provides more detail.

* **end**() - The socket was disconnected.

* **close**() - The socket was closed.

#### Client methods

* **(constructor)**() - Creates and returns a new Client instance.

* **connect**(< _object_ >config) - _(void)_ - Attempts a connection to a server using the information given in `config`:

    * **host** - _string_ - Hostname or IP address of the server. **Default:** `'localhost'`

    * **port** - _integer_ - Port number of the server. **Default:** `22`

    * **localAddress** - _string_ - IP address of the network interface to use to connect to the server. **Default:** (none -- determined by OS)

    * **localPort** - _string_ - The local port number to connect from. **Default:** (none -- determined by OS)

    * **forceIPv4** - _boolean_ - Only connect via resolved IPv4 address for `host`. **Default:** `false`

    * **forceIPv6** - _boolean_ - Only connect via resolved IPv6 address for `host`. **Default:** `false`

    * **hostHash** - _string_ - Any valid hash algorithm supported by node. The host's key is hashed using this algorithm and passed to the **hostVerifier** function as a hex string. **Default:** (none)

    * **hostVerifier** - _function_ - Function with parameters `(hashedKey[, callback])` where `hashedKey` is a string hex hash of the host's key for verification purposes. Return `true` to continue with the handshake or `false` to reject and disconnect, or call `callback()` with `true` or `false` if you need to perform asynchronous verification. **Default:** (auto-accept if `hostVerifier` is not set)

    * **username** - _string_ - Username for authentication. **Default:** (none)

    * **password** - _string_ - Password for password-based user authentication. **Default:** (none)

    * **agent** - _string_ - Path to ssh-agent's UNIX socket for ssh-agent-based user authentication. **Windows users: set to 'pageant' for authenticating with Pageant or (actual) path to a cygwin "UNIX socket."** **Default:** (none)

    * **agentForward** - _boolean_ - Set to `true` to use OpenSSH agent forwarding (`auth-agent@openssh.com`) for the life of the connection. `agent` must also be set to use this feature. **Default:** `false`

    * **privateKey** - _mixed_ - _Buffer_ or _string_ that contains a private key for either key-based or hostbased user authentication (OpenSSH format). **Default:** (none)

    * **passphrase** - _string_ - For an encrypted private key, this is the passphrase used to decrypt it. **Default:** (none)

    * **localHostname** - _string_ - Along with **localUsername** and **privateKey**, set this to a non-empty string for hostbased user authentication. **Default:** (none)

    * **localUsername** - _string_ - Along with **localHostname** and **privateKey**, set this to a non-empty string for hostbased user authentication. **Default:** (none)

    * **tryKeyboard** - _boolean_ - Try keyboard-interactive user authentication if primary user authentication method fails. If you set this to `true`, you need to handle the `keyboard-interactive` event. **Default:** `false`

    * **authHandler** - _function_ - Function with parameters `(methodsLeft, partialSuccess, callback)` where `methodsLeft` and `partialSuccess` are `null` on the first authentication attempt, otherwise are an array and boolean respectively. Return or call `callback()` with the name of the authentication method to try next (pass `false` to signal no more methods to try). Valid method names are: `'none', 'password', 'publickey', 'agent', 'keyboard-interactive', 'hostbased'`. **Default:** function that follows a set method order: None -> Password -> Private Key -> Agent (-> keyboard-interactive if `tryKeyboard` is `true`) -> Hostbased

    * **keepaliveInterval** - _integer_ - How often (in milliseconds) to send SSH-level keepalive packets to the server (in a similar way as OpenSSH's ServerAliveInterval config option). Set to 0 to disable. **Default:** `0`

    * **keepaliveCountMax** - _integer_ - How many consecutive, unanswered SSH-level keepalive packets that can be sent to the server before disconnection (similar to OpenSSH's ServerAliveCountMax config option). **Default:** `3`

    * **readyTimeout** - _integer_ - How long (in milliseconds) to wait for the SSH handshake to complete. **Default:** `20000`

    * **sock** - _ReadableStream_ - A _ReadableStream_ to use for communicating with the server instead of creating and using a new TCP connection (useful for connection hopping).

    * **strictVendor** - _boolean_ - Performs a strict server vendor check before sending vendor-specific requests, etc. (e.g. check for OpenSSH server when using `openssh_noMoreSessions()`) **Default:** `true`

    * **algorithms** - _object_ - This option allows you to explicitly override the default transport layer algorithms used for the connection. The value for each category must either be an array of valid algorithm names or an object containing `append`, `prepend`, and/or `remove` properties that each contain an _array_ of algorithm names or RegExps to match to adjust default lists for each category. For arrays, the order of the algorithm names matters, with the most favorable being first. Valid keys:

        * **kex** - _mixed_ - Key exchange algorithms.
            * Default list (in order from most to least preferrable):
              * `curve25519-sha256 (node v14.0.0+)`
              * `curve25519-sha256@libssh.org (node v14.0.0+)`
              * `ecdh-sha2-nistp256`
              * `ecdh-sha2-nistp384`
              * `ecdh-sha2-nistp521`
              * `diffie-hellman-group-exchange-sha256`
              * `diffie-hellman-group14-sha256`
              * `diffie-hellman-group15-sha512`
              * `diffie-hellman-group16-sha512`
              * `diffie-hellman-group17-sha512`
              * `diffie-hellman-group18-sha512`
            * Other supported names:
              * `diffie-hellman-group-exchange-sha1`
              * `diffie-hellman-group14-sha1`
              * `diffie-hellman-group1-sha1`

        * **serverHostKey** - _mixed_ - Server host key formats.
            * Default list (in order from most to least preferrable):
              * `ssh-ed25519` (node v12.0.0+)
              * `ecdsa-sha2-nistp256`
              * `ecdsa-sha2-nistp384`
              * `ecdsa-sha2-nistp521`
              * `rsa-sha2-512`
              * `rsa-sha2-256`
              * `ssh-rsa`
            * Other supported names:
              * `ssh-dss`

        * **cipher** - _mixed_ - Ciphers.
            * Default list (in order from most to least preferrable):
              * `chacha20-poly1305@openssh.com` (priority of chacha20-poly1305 may vary depending upon CPU and/or optional binding availability)
              * `aes128-gcm`
              * `aes128-gcm@openssh.com`
              * `aes256-gcm`
              * `aes256-gcm@openssh.com`
              * `aes128-ctr`
              * `aes192-ctr`
              * `aes256-ctr`
            * Other supported names:
              * `3des-cbc`
              * `aes256-cbc`
              * `aes192-cbc`
              * `aes128-cbc`
              * `arcfour256`
              * `arcfour128`
              * `arcfour`
              * `blowfish-cbc`
              * `cast128-cbc`

        * **hmac** - _mixed_ - (H)MAC algorithms.
            * Default list (in order from most to least preferrable):
              * `hmac-sha2-256-etm@openssh.com`
              * `hmac-sha2-512-etm@openssh.com`
              * `hmac-sha1-etm@openssh.com`
              * `hmac-sha2-256`
              * `hmac-sha2-512`
              * `hmac-sha1`
            * Other supported names:
              * hmac-md5
              * hmac-sha2-256-96
              * hmac-sha2-512-96
              * hmac-ripemd160
              * hmac-sha1-96
              * hmac-md5-96

        * **compress** - _mixed_ - Compression algorithms.
            * Default list (in order from most to least preferrable):
              * `none`
              * `zlib@openssh.com`
              * `zlib`
            * Other supported names:


    * **debug** - _function_ - Set this to a function that receives a single string argument to get detailed (local) debug information. **Default:** (none)

* **exec**(< _string_ >command[, < _object_ >options], < _function_ >callback) - _(void)_ - Executes `command` on the server. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream. Valid `options` properties are:

    * **env** - _object_ - An environment to use for the execution of the command.

    * **pty** - _mixed_ - Set to `true` to allocate a pseudo-tty with defaults, or an object containing specific pseudo-tty settings (see 'Pseudo-TTY settings'). Setting up a pseudo-tty can be useful when working with remote processes that expect input from an actual terminal (e.g. sudo's password prompt).

    * **x11** - _mixed_ - Set to `true` to use defaults below, set to a number to specify a specific screen number, or an object with the following valid properties:

        * **single** - _boolean_ - Allow just a single connection? **Default:** `false`

        * **screen** - _number_ - Screen number to use **Default:** `0`

        * **protocol** - _string_ - The authentication protocol name. **Default:** `'MIT-MAGIC-COOKIE-1'`

        * **cookie** - _mixed_ - The authentication cookie. Can be a hex _string_ or a _Buffer_ containing the raw cookie value (which will be converted to a hex string). **Default:** (random 16 byte value)

* **shell**([[< _mixed_ >window,] < _object_ >options]< _function_ >callback) - _(void)_ - Starts an interactive shell session on the server, with an optional `window` object containing pseudo-tty settings (see 'Pseudo-TTY settings'). If `window === false`, then no pseudo-tty is allocated. `options` supports the `x11` and `env` options as described in `exec()`. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **forwardIn**(< _string_ >remoteAddr, < _integer_ >remotePort, < _function_ >callback) - _(void)_ - Bind to `remoteAddr` on `remotePort` on the server and forward incoming TCP connections. `callback` has 2 parameters: < _Error_ >err, < _integer_ >port (`port` is the assigned port number if `remotePort` was 0). Here are some special values for `remoteAddr` and their associated binding behaviors:

    * '' - Connections are to be accepted on all protocol families supported by the server.

    * '0.0.0.0' - Listen on all IPv4 addresses.

    * '::' - Listen on all IPv6 addresses.

    * 'localhost' - Listen on all protocol families supported by the server on loopback addresses only.

    * '127.0.0.1' and '::1' - Listen on the loopback interfaces for IPv4 and IPv6, respectively.

* **unforwardIn**(< _string_ >remoteAddr, < _integer_ >remotePort, < _function_ >callback) - _(void)_ - Unbind from `remoteAddr` on `remotePort` on the server and stop forwarding incoming TCP connections. Until `callback` is called, more connections may still come in. `callback` has 1 parameter: < _Error_ >err.

* **forwardOut**(< _string_ >srcIP, < _integer_ >srcPort, < _string_ >dstIP, < _integer_ >dstPort, < _function_ >callback) - _(void)_ - Open a connection with `srcIP` and `srcPort` as the originating address and port and `dstIP` and `dstPort` as the remote destination address and port. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **sftp**(< _function_ >callback) - _(void)_ - Starts an SFTP session. `callback` has 2 parameters: < _Error_ >err, < _SFTP_ >sftp. For methods available on `sftp`, see the [`SFTP` client documentation](https://github.com/mscdex/ssh2/blob/master/SFTP.md).

* **subsys**(< _string_ >subsystem, < _function_ >callback) - _(void)_ - Invokes `subsystem` on the server. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **rekey**([< _function_ >callback]) - _(void)_ - Initiates a rekey with the server. If `callback` is supplied, it is added as a one-time handler for the `rekey` event.

* **end**() - _(void)_ - Disconnects the socket.

* **openssh_noMoreSessions**(< _function_ >callback) - _(void)_ - OpenSSH extension that sends a request to reject any new sessions (e.g. exec, shell, sftp, subsys) for this connection. `callback` has 1 parameter: < _Error_ >err.

* **openssh_forwardInStreamLocal**(< _string_ >socketPath, < _function_ >callback) - _(void)_ - OpenSSH extension that binds to a UNIX domain socket at `socketPath` on the server and forwards incoming connections. `callback` has 1 parameter: < _Error_ >err.

* **openssh_unforwardInStreamLocal**(< _string_ >socketPath, < _function_ >callback) - _(void)_ - OpenSSH extension that unbinds from a UNIX domain socket at `socketPath` on the server and stops forwarding incoming connections. `callback` has 1 parameter: < _Error_ >err.

* **openssh_forwardOutStreamLocal**(< _string_ >socketPath, < _function_ >callback) - _(void)_ - OpenSSH extension that opens a connection to a UNIX domain socket at `socketPath` on the server. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

### Server

#### Server events

* **connection**(< _Connection_ >client, < _object_ >info) - A new client has connected. `info` contains the following properties:

    * **ip** - _string_ - The `remoteAddress` of the connection.

    * **family** - _string_ - The `remoteFamily` of the connection.

    * **port** - _integer_ - The `remotePort` of the connection.

    * **header** - _object_ - Information about the client's header:

        * **identRaw** - _string_ - The raw client identification string.

        * **versions** - _object_ - Various version information:

            * **protocol** - _string_ - The SSH protocol version (always `1.99` or `2.0`).

            * **software** - _string_ - The software name and version of the client.

        * **comments** - _string_ - Any text that comes after the software name/version.

    Example: the identification string `SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2` would be parsed as:

```js
        { identRaw: 'SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2',
          version: {
            protocol: '2.0',
            software: 'OpenSSH_6.6.1p1'
          },
          comments: 'Ubuntu-2ubuntu2' }
```

#### Server methods

* **(constructor)**(< _object_ >config[, < _function_ >connectionListener]) - Creates and returns a new Server instance. Server instances also have the same methods/properties/events as [`net.Server`](http://nodejs.org/docs/latest/api/net.html#net_class_net_server). `connectionListener` if supplied, is added as a `connection` listener. Valid `config` properties:

    * **hostKeys** - _array_ - An array of either Buffers/strings that contain host private keys or objects in the format of `{ key: <Buffer/string>, passphrase: <string> }` for encrypted private keys. (**Required**) **Default:** (none)

    * **algorithms** - _object_ - This option allows you to explicitly override the default transport layer algorithms used for incoming client connections. Each value must be an array of valid algorithms for that category. The order of the algorithms in the arrays are important, with the most favorable being first. For a list of valid and default algorithm names, please review the documentation for the version of `ssh2` used by this module. Valid keys:

        * **kex** - _array_ - Key exchange algorithms.

        * **cipher** - _array_ - Ciphers.

        * **serverHostKey** - _array_ - Server host key formats.

        * **hmac** - _array_ - (H)MAC algorithms.

        * **compress** - _array_ - Compression algorithms.

    * **greeting** - _string_ - A message that is sent to clients immediately upon connection, before handshaking begins. **Note:** Most clients usually ignore this. **Default:** (none)

    * **banner** - _string_ - A message that is sent to clients once, right before authentication begins. **Default:** (none)

    * **ident** - _string_ - A custom server software name/version identifier. **Default:** `'ssh2js' + moduleVersion + 'srv'`

    * **highWaterMark** - _integer_ - This is the `highWaterMark` to use for the parser stream. **Default:** `32 * 1024`

    * **debug** - _function_ - Set this to a function that receives a single string argument to get detailed (local) debug information. **Default:** (none)

#### Connection events

* **authentication**(< _AuthContext_ >ctx) - The client has requested authentication. `ctx.username` contains the client username, `ctx.method` contains the requested authentication method, and `ctx.accept()` and `ctx.reject([< Array >authMethodsLeft[, < Boolean >isPartialSuccess]])` are used to accept or reject the authentication request respectively. `abort` is emitted if the client aborts the authentication request. Other properties/methods available on `ctx` depends on the `ctx.method` of authentication the client has requested:

    * `password`:

        * **password** - _string_ - This is the password sent by the client.

    * `publickey`:

        * **key** - _object_ - Contains information about the public key sent by the client:

            * **algo** - _string_ - The name of the key algorithm (e.g. `ssh-rsa`).

            * **data** - _Buffer_ - The actual key data.

        * **sigAlgo** - _mixed_ - If the value is `undefined`, the client is only checking the validity of the `key`. If the value is a _string_, then this contains the signature algorithm that is passed to [`crypto.createVerify()`](http://nodejs.org/docs/latest/api/crypto.html#crypto_crypto_createverify_algorithm).

        * **blob** - _mixed_ - If the value is `undefined`, the client is only checking the validity of the `key`. If the value is a _Buffer_, then this contains the data that is passed to [`verifier.update()`](http://nodejs.org/docs/latest/api/crypto.html#crypto_verifier_update_data).

        * **signature** - _mixed_ - If the value is `undefined`, the client is only checking the validity of the `key`. If the value is a _Buffer_, then this contains a signature that is passed to [`verifier.verify()`](http://nodejs.org/docs/latest/api/crypto.html#crypto_verifier_verify_object_signature_signature_format).

    * `keyboard-interactive`:

        * **submethods** - _array_ - A list of preferred authentication "sub-methods" sent by the client. This may be used to determine what (if any) prompts to send to the client.

        * **prompt**(< _array_ >prompts[, < _string_ >title[, < _string_ >instructions]], < _function_ >callback) - _(void)_ - Send prompts to the client. `prompts` is an array of `{ prompt: 'Prompt text', echo: true }` objects (`prompt` being the prompt text and `echo` indicating whether the client's response to the prompt should be echoed to their display). `callback` is called with `(err, responses)`, where `responses` is an array of string responses matching up to the `prompts`.

* **ready**() - Emitted when the client has been successfully authenticated.

* **session**(< _function_ >accept, < _function_ >reject) - Emitted when the client has requested a new session. Sessions are used to start interactive shells, execute commands, request X11 forwarding, etc. `accept()` returns a new _Session_ instance.

* **tcpip**(< _function_ >accept, < _function_ >reject, < _object_ >info) - Emitted when the client has requested an outbound (TCP) connection. `accept()` returns a new _Channel_ instance representing the connection. `info` contains:

    * **srcIP** - _string_ - Source IP address of outgoing connection.

    * **srcPort** - _string_ - Source port of outgoing connection.

    * **destIP** - _string_ - Destination IP address of outgoing connection.

    * **destPort** - _string_ - Destination port of outgoing connection.

* **openssh.streamlocal**(< _function_ >accept, < _function_ >reject, < _object_ >info) - Emitted when the client has requested a connection to a UNIX domain socket. `accept()` returns a new _Channel_ instance representing the connection. `info` contains:

    * **socketPath** - _string_ - Destination socket path of outgoing connection.

* **request**(< _mixed_ >accept, < _mixed_ >reject, < _string_ >name, < _object_ >info) - Emitted when the client has sent a global request for `name` (e.g. `tcpip-forward` or `cancel-tcpip-forward`). `accept` and `reject` are functions if the client requested a response. If `bindPort === 0`, you should pass the chosen port to `accept()` so that the client will know what port was bound. `info` contains additional details about the request:

    * `tcpip-forward` and `cancel-tcpip-forward`:

        * **bindAddr** - _string_ - The IP address to start/stop binding to.

        * **bindPort** - _integer_ - The port to start/stop binding to.

    * `streamlocal-forward@openssh.com` and `cancel-streamlocal-forward@openssh.com`:

        * **socketPath** - _string_ - The socket path to start/stop binding to.

* **handshake**(< _object_ >negotiated) - Emitted when a handshake has completed (either initial or rekey). `negotiated` contains the negotiated details of the handshake and is of the form:

```js
    // In this particular case `mac` is empty because there is no separate MAC
    // because it's integrated into AES in GCM mode
    { kex: 'ecdh-sha2-nistp256',
      srvHostKey: 'rsa-sha2-512',
      cs: { // Client to server algorithms
        cipher: 'aes128-gcm',
        mac: '',
        compress: 'none',
        lang: ''
      },
      sc: { // Server to client algorithms
        cipher: 'aes128-gcm',
        mac: '',
        compress: 'none',
        lang: ''
      }
    }
```   

* **rekey**() - Emitted when a rekeying operation has completed (either client or server-initiated).

* **error**(< _Error_ >err) - An error occurred.

* **end**() - The client socket disconnected.

* **close**() - The client socket was closed.

#### Connection methods

* **end**() - _(void)_ - Closes the client connection.

* **x11**(< _string_ >originAddr, < _integer_ >originPort, < _function_ >callback) - _(void)_ - Alert the client of an incoming X11 client connection from `originAddr` on port `originPort`. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **forwardOut**(< _string_ >boundAddr, < _integer_ >boundPort, < _string_ >remoteAddr, < _integer_ >remotePort, < _function_ >callback) - _(void)_ - Alert the client of an incoming TCP connection on `boundAddr` on port `boundPort` from `remoteAddr` on port `remotePort`. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **openssh_forwardOutStreamLocal**(< _string_ >socketPath, < _function_ >callback) - _(void)_ - Alert the client of an incoming UNIX domain socket connection on `socketPath`. `callback` has 2 parameters: < _Error_ >err, < _Channel_ >stream.

* **rekey**([< _function_ >callback]) - _(void)_ - Initiates a rekey with the client. If `callback` is supplied, it is added as a one-time handler for the `rekey` event.

#### Session events

* **pty**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client requested allocation of a pseudo-TTY for this session. `accept` and `reject` are functions if the client requested a response. `info` has these properties:

    * **cols** - _integer_ - The number of columns for the pseudo-TTY.

    * **rows** - _integer_ - The number of rows for the pseudo-TTY.

    * **width** - _integer_ - The width of the pseudo-TTY in pixels.

    * **height** - _integer_ - The height of the pseudo-TTY in pixels.

    * **modes** - _object_ - Contains the requested terminal modes of the pseudo-TTY keyed on the mode name with the value being the mode argument. (See the table at the end for valid names).

* **window-change**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client reported a change in window dimensions during this session. `accept` and `reject` are functions if the client requested a response. `info` has these properties:

    * **cols** - _integer_ - The new number of columns for the client window.

    * **rows** - _integer_ - The new number of rows for the client window.

    * **width** - _integer_ - The new width of the client window in pixels.

    * **height** - _integer_ - The new height of the client window in pixels.

* **x11**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client requested X11 forwarding. `accept` and `reject` are functions if the client requested a response. `info` has these properties:

    * **single** - _boolean_ - `true` if only a single connection should be forwarded.

    * **protocol** - _string_ - The name of the X11 authentication method used (e.g. `MIT-MAGIC-COOKIE-1`).

    * **cookie** - _string_ - The X11 authentication cookie encoded in hexadecimal.

    * **screen** - _integer_ - The screen number to forward X11 connections for.

* **env**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client requested an environment variable to be set for this session. `accept` and `reject` are functions if the client requested a response. `info` has these properties:

    * **key** - _string_ - The environment variable's name.

    * **value** - _string_ - The environment variable's value.

* **signal**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client has sent a signal. `accept` and `reject` are functions if the client requested a response. `info` has these properties:

    * **name** - _string_ - The signal name (e.g. `SIGUSR1`).

* **auth-agent**(< _mixed_ >accept, < _mixed_ >reject) - The client has requested incoming ssh-agent requests be forwarded to them. `accept` and `reject` are functions if the client requested a response.

* **shell**(< _mixed_ >accept, < _mixed_ >reject) - The client has requested an interactive shell. `accept` and `reject` are functions if the client requested a response. `accept()` returns a _Channel_ for the interactive shell.

* **exec**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client has requested execution of a command string. `accept` and `reject` are functions if the client requested a response. `accept()` returns a _Channel_ for the command execution. `info` has these properties:

    * **command** - _string_ - The command line to be executed.

* **sftp**(< _mixed_ >accept, < _mixed_ >reject) - The client has requested the SFTP subsystem. `accept` and `reject` are functions if the client requested a response. `accept()` returns an _SFTP_ instance in server mode (see the [`SFTP` documentation](https://github.com/mscdex/ssh2/blob/master/SFTP.md) for details). `info` has these properties:

* **subsystem**(< _mixed_ >accept, < _mixed_ >reject, < _object_ >info) - The client has requested an arbitrary subsystem. `accept` and `reject` are functions if the client requested a response. `accept()` returns a _Channel_ for the subsystem. `info` has these properties:

    * **name** - _string_ - The name of the subsystem.

* **close**() - The session was closed.

### Channel

This is a normal **streams2** Duplex Stream (used both by clients and servers), with the following changes:

* A boolean property `allowHalfOpen` exists and behaves similarly to the property of the same name for `net.Socket`. When the stream's end() is called, if `allowHalfOpen` is `true`, only EOF will be sent (the server can still send data if they have not already sent EOF). The default value for this property is `true`.

* A `close` event is emitted once the channel is completely closed on both the client and server.

* Client-specific:

    * For exec():

        * An `exit` event *may* (the SSH2 spec says it is optional) be emitted when the process finishes. If the process finished normally, the process's return value is passed to the `exit` callback. If the process was interrupted by a signal, the following are passed to the `exit` callback: null, < _string_ >signalName, < _boolean_ >didCoreDump, < _string_ >description.

        * If there was an `exit` event, the `close` event will be passed the same arguments for convenience.

        * A `stderr` property contains a Readable stream that represents output from stderr.

    * For shell() and exec():

        * The readable side represents stdout and the writable side represents stdin.

        * **signal**(< _string_ >signalName) - _(void)_ - Sends a POSIX signal to the current process on the server. Valid signal names are: 'ABRT', 'ALRM', 'FPE', 'HUP', 'ILL', 'INT', 'KILL', 'PIPE', 'QUIT', 'SEGV', 'TERM', 'USR1', and 'USR2'. Some server implementations may ignore this request if they do not support signals. Note: If you are trying to send SIGINT and you find `signal()` doesn't work, try writing `'\x03'` to the Channel stream instead.

        * **setWindow**(< _integer_ >rows, < _integer_ >cols, < _integer_ >height, < _integer_ >width) - _(void)_ - Lets the server know that the local terminal window has been resized. The meaning of these arguments are described in the 'Pseudo-TTY settings' section.

* Server-specific:

    * For exec-enabled channel instances there is an additional method available that may be called right before you close the channel. It has two different signatures:

        * **exit**(< _integer_ >exitCode) - _(void)_ - Sends an exit status code to the client.

        * **exit**(< _string_ >signalName[, < _boolean_ >coreDumped[, < _string_ >errorMsg]]) - _(void)_ - Sends an exit status code to the client.

    * For exec and shell-enabled channel instances, `channel.stderr` is a writable stream.

### Pseudo-TTY settings

* **rows** - < _integer_ > - Number of rows. **Default:** `24`

* **cols** - < _integer_ > - Number of columns. **Default:** `80`

* **height** - < _integer_ > - Height in pixels. **Default:** `480`

* **width** - < _integer_ > - Width in pixels. **Default:** `640`

* **term** - < _string_ > - The value to use for $TERM. **Default:** `'vt100'`

* **modes** - < _object_ > - An object containing [Terminal Modes](#terminal-modes) as keys, with each value set to each mode argument. **Default:** `null`

`rows` and `cols` override `width` and `height` when `rows` and `cols` are non-zero.

Pixel dimensions refer to the drawable area of the window.

Zero dimension parameters are ignored.

### Terminal modes

Name           | Description
-------------- | ------------
VINTR          | Interrupt character; 255 if none. Similarly for the other characters. Not all of these characters are supported on all systems.
VQUIT          | The quit character (sends SIGQUIT signal on POSIX systems).
VERASE         | Erase the character to left of the cursor.
VKILL          | Kill the current input line.
VEOF           | End-of-file character (sends EOF from the terminal).
VEOL           | End-of-line character in addition to carriage return and/or linefeed.
VEOL2          | Additional end-of-line character.
VSTART         | Continues paused output (normally control-Q).
VSTOP          | Pauses output (normally control-S).
VSUSP          | Suspends the current program.
VDSUSP         | Another suspend character.
VREPRINT       | Reprints the current input line.
VWERASE        | Erases a word left of cursor.
VLNEXT         | Enter the next character typed literally, even if it is a special character
VFLUSH         | Character to flush output.
VSWTCH         | Switch to a different shell layer.
VSTATUS        | Prints system status line (load, command, pid, etc).
VDISCARD       | Toggles the flushing of terminal output.
IGNPAR         | The ignore parity flag. The parameter SHOULD be 0 if this flag is FALSE, and 1 if it is TRUE.
PARMRK         | Mark parity and framing errors.
INPCK          | Enable checking of parity errors.
ISTRIP         | Strip 8th bit off characters.
INLCR          | Map NL into CR on input.
IGNCR          | Ignore CR on input.
ICRNL          | Map CR to NL on input.
IUCLC          | Translate uppercase characters to lowercase.
IXON           | Enable output flow control.
IXANY          | Any char will restart after stop.
IXOFF          | Enable input flow control.
IMAXBEL        | Ring bell on input queue full.
ISIG           | Enable signals INTR, QUIT, [D]SUSP.
ICANON         | Canonicalize input lines.
XCASE          | Enable input and output of uppercase characters by preceding their lowercase equivalents with "\".
ECHO           | Enable echoing.
ECHOE          | Visually erase chars.
ECHOK          | Kill character discards current line.
ECHONL         | Echo NL even if ECHO is off.
NOFLSH         | Don't flush after interrupt.
TOSTOP         | Stop background jobs from output.
IEXTEN         | Enable extensions.
ECHOCTL        | Echo control characters as ^(Char).
ECHOKE         | Visual erase for line kill.
PENDIN         | Retype pending input.
OPOST          | Enable output processing.
OLCUC          | Convert lowercase to uppercase.
ONLCR          | Map NL to CR-NL.
OCRNL          | Translate carriage return to newline (output).
ONOCR          | Translate newline to carriage return-newline (output).
ONLRET         | Newline performs a carriage return (output).
CS7            | 7 bit mode.
CS8            | 8 bit mode.
PARENB         | Parity enable.
PARODD         | Odd parity, else even.
TTY_OP_ISPEED  | Specifies the input baud rate in bits per second.
TTY_OP_OSPEED  | Specifies the output baud rate in bits per second.

### HTTPAgent

#### HTTPAgent methods

* **(constructor)**(< _object_ >sshConfig[, < _object_ >agentConfig]) - Creates and returns a new `http.Agent` instance used to tunnel an HTTP connection over SSH. `sshConfig` is what is passed to `client.connect()` and `agentOptions` is passed to the `http.Agent` constructor.

### HTTPSAgent

#### HTTPSAgent methods

* **(constructor)**(< _object_ >sshConfig[, < _object_ >agentConfig]) - Creates and returns a new `https.Agent` instance used to tunnel an HTTP connection over SSH. `sshConfig` is what is passed to `client.connect()` and `agentOptions` is passed to the `https.Agent` constructor.

### Utilities

* **parseKey**(< _mixed_ >keyData[, < _string_ >passphrase]) - _mixed_ - Parses a private/public key in OpenSSH, RFC4716, or PPK format. For encrypted private keys, the key will be decrypted with the given `passphrase`. `keyData` can be a _Buffer_ or _string_ value containing the key contents. The returned value will be an array of objects (currently in the case of modern OpenSSH keys) or an object with these properties and methods:

    * **type** - _string_ - The full key type (e.g. `'ssh-rsa'`)

    * **comment** - _string_ - The comment for the key

    * **getPrivatePEM**() - _string_ - This returns the PEM version of a private key

    * **getPublicPEM**() - _string_ - This returns the PEM version of a public key (for either public key or derived from a private key)

    * **getPublicSSH**() - _string_ - This returns the SSH version of a public key (for either public key or derived from a private key)

    * **sign**(< _mixed_ >data) - _mixed_ - This signs the given `data` using this key and returns a _Buffer_ containing the signature on success. On failure, an _Error_ will be returned. `data` can be anything accepted by node's [`sign.update()`](https://nodejs.org/docs/latest/api/crypto.html#crypto_sign_update_data_inputencoding).

    * **verify**(< _mixed_ >data, < _Buffer_ >signature) - _mixed_ - This verifies a `signature` of the given `data` using this key and returns `true` if the signature could be verified. On failure, either `false` will be returned or an _Error_ will be returned upon a more critical failure. `data` can be anything accepted by node's [`verify.update()`](https://nodejs.org/docs/latest/api/crypto.html#crypto_verify_update_data_inputencoding).

* **sftp.OPEN_MODE** - [`OPEN_MODE`](https://github.com/mscdex/ssh2/blob/master/SFTP.md#useful-standalone-data-structures)

* **sftp.STATUS_CODE** - [`STATUS_CODE`](https://github.com/mscdex/ssh2/blob/master/SFTP.md#useful-standalone-data-structures)

* **sftp.flagsToString** - [`flagsToString()`](https://github.com/mscdex/ssh2/blob/master/SFTP.md#useful-standalone-methods)

* **sftp.stringToFlags** - [`stringToFlags()`](https://github.com/mscdex/ssh2/blob/master/SFTP.md#useful-standalone-methods)
