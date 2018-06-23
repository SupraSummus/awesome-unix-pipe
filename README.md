# Awesome UNIX pipe

Rules:
 * all connections to outside world are file descriptors (no files, no network, no UNIX domain addresses)
 * no sockets, except if they can be modeled with pipes

## Basic operations

### "Constant" stream

    +----------------------+
    | `echo -n 'content!'` |
    |                 FD:1 | -> stream containing "content!"
    +----------------------+

### Concatenation

                 +--------------+
                 | concatenator |
    data ------> | FD:0    FD:1 | -> data + more data
    more data -> | FD:3         |
                 +--------------+

| concatenator | comments |
| ------------ | -------- |
| `cat /proc/self/fd/0 /proc/self/fd/3` | |
| `pv -q /proc/self/fd/0 /proc/self/fd/3` | |

### Multiplexing

Number of multiplexed streams is constat at runtime.

              +-----------+                  +-----------+
              | muxer     |                  | demuxer   |
    data a -> | FD:0 FD:1 | -> mixed data -> | FD:0 FD:1 | -> data a
    data b -> | FD:3      |                  |      FD:3 | -> data b
    ...       +-----------+                  +-----------+    ...

Haven't found this one yet ;)

### Bonds

Using multiple pipes to transmit single stream.

Bonds can provide more throughput and/or redundancy.

            +-----------+                +-----------+
            | bond      |                | unbond    |
    data -> | FD:0 FD:1 | -> stream a -> | FD:0 FD:1 | -> data
            |      FD:3 | -> stream b -> | FD:3      |
            +-----------+    ...         +-----------+

Haven't found this one too.

## Buffers

Add buffer on the pipe. UNIX pipes are buffered but sometimes extra space is needed. In this example buffer size is 1GiB.

            +-----------+
            | buffer    |
    data -> | FD:0 FD:1 | -> data
            +-----------+

| buffer | comments |
| ------ | -------- |
| `pv -q -B 1G` | |

## Throughput metrics

Measure parameters of pipe throughput. Output human-readable data.

            +-----------+
            | meter     |
    data -> | FD:0 FD:1 | -> data
            |      FD:2 | -> mesurement data
            +-----------+

| meter | comments |
| ----- | -------- |
| `pv` | Has some switches to customize output |

## Compression

            +------------+                       +--------------+
            | compressor |                       | decompressor |
    data -> | FD:0  FD:1 | -> compressed data -> | FD:0    FD:1 | -> data
            +------------+                       +--------------+

| compressor | decompressor | comments |
| ---------- | ------------ | -------- |
| `gzip`     | `gzip -d`    |          |

## Cryptography

                      +-----------+                      +-----------+
                      | encryptor |                      | decryptor |
    data -----------> | FD:0 FD:1 | -> encrypted data -> | FD:0 FD:1 | -> data
    encryption key -> | FD:3      |    decryption key -> | FD:3      |
                      +-----------+                      +-----------+

### Symmetric

Encryption key and decryption key are the same.

#### Unauthenticated

These has no protection against modifying transmitted data.

| encryptor | decryptor | comments |
| --------- | --------- | -------- |
| `openssl enc -e -keyfile /proc/self/fd/3` | `openssl enc -d -keyfile /proc/self/fd/3` | |
| `aespipe -p 3` | `aespipe -p 3 -d` | Has some requirements for key format. Weak. |
| | | [private-pipe](https://github.com/emilbayes/private-pipe) |

#### Authenticated

These will detect if transmitted data was modified.

| encryptor | decryptor | comments |
| --------- | --------- | -------- |
| `gpg --symmetric --no-use-agent --passphrase-file /proc/self/fd/3 --cipher-algo AES256 -z0` | `gpg --decrypt --no-use-agent --passphrase-file /proc/self/fd/3` | |
| `aepipe /proc/self/fd/3` | `aepipe -d /proc/self/fd/3` | part of [keypipe](https://github.com/hashbrowncipher/keypipe) software |
| | | [blowpipe](https://github.com/skeeto/blowpipe) |

### Asymmetric

| encryptor | decryptor | comments |
| --------- | --------- | -------- |
| | | [rsa-stream](https://github.com/substack/rsa-stream) |

## Maths

### Bitwise operations

              +-----------+
              | function  |
    data a -> | FD:0 FD:1 | -> bitwise function(data a, data b, ...)
    data b -> | FD:3      |
    ...       +-----------+

Haven't found anything.

# Interfaces to non-UNIX-pipe media

 * virtual network tunnels
   * [socat](http://www.dest-unreach.org/socat/doc/socat-tun.html)
 * TCP
   * client: `netcat -c HOST PORT` (GNU netcat)
   * server: `netcat -l -p PORT` (GNU netcat)
   * socat
 * HTTP(S)
   * client: `curl`
 * websocket
   * [websocketd](https://github.com/joewalnes/websocketd)
   * [wscat](https://github.com/websockets/wscat)
