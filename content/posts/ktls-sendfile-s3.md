+++
title = "Uploading a file to S3 without reading it"
date = 2025-08-03
taxonomies.tags = ["optimization", "Erlang"]

[extra]
repo_view = true
+++

This post covers uploading a file to S3. It's pretty simple: make an HTTP/1.1 `PUT` request and now your file is in _the cloud_ 🪄.

What could possibly be interesting about something so mundane? Well, we're going to leverage the Linux kernel to do it in an unnecessarily efficient way. (Note: I think FreeBSD also supports this but I haven't tried it.)

We'll be looking at a novel way - used by no existing AWS SDK clients to my knowledge - to upload to S3.

## How you'd normally do it

AWS S3 has a REST HTTP API which accepts HTTP/1.1. You can add an "object" (basically a file) to the "bucket" (kinda like a file system) with a [PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) request.

HTTP/1.1 is super simple. To quote [RFC 7230](https://www.rfc-editor.org/rfc/rfc7230), the syntax looks like so:

```
HTTP-message   = start-line
                 *( header-field CRLF )
                 CRLF
                 [ message-body ]
```

For example inserting an object could look like this (quoting the AWS documentation):

```
PUT /example-object HTTP/1.1
Host: example-bucket.s3.<Region>.amazonaws.com
Accept: */*
Authorization: <authorization string>
Date: Thu, 22 Sep 2016 21:58:13 GMT
x-amz-tagging: tag1=value1&tag2=value2

[... bytes of object data] 
```

The `start-line` is the HTTP method, path, and "HTTP/1.1" all separated by spaces. Then you have a set of headers. Then an empty line and then the body. All of these lines use the CRLF line-ending (`\r\n`). Open a TLS socket to this S3 endpoint and send that binary data (after TLS encryption) and you can upload to S3.

> A quick aside: calculating the authorization header is a little bit involved. I will totally skip over it in this post. The proper docs are [here](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-header-based-auth.html) and a simple Erlang implementation is included in the gist linked later.

In Erlang this might look like so:

```erlang
upload(File) ->
    {ok, Socket} = ssl:connect("s3.us-east-2.amazonaws.com", 443, []),
    %% Send an HTTP/1.1 header based on the file's size (content-length header)
    %% and auth parameters like the secret access key.
    ok = ssl:send(Socket, http1_header(File)),
    %% Read the file into memory.
    {ok, FileContents} = file:read_file(File),
    %% And then send it on the socket, as the body.
    ok = ssl:send(Socket, FileContents),
    %% Receiving a response is overrated. Let's just close the socket.
    ssl:close(Socket).
```

Erlang has an odd Prolog-like syntax but hopefully you get the gist. We open a connection to S3 and send an HTTP/1.1 PUT request with the file's contents as the HTTP/1.1 body.

This is nice and straightforward but it means that we need to read the entire file contents into memory in order to upload the file. We could read in smaller chunks or use a chunked transfer-encoding or even a fancy [multi-part upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html). No matter how we slice it though, we need to read the file into our Erlang program's memory.

Or do we?!

## sendfile(2)

`sendfile(2)` is a system call which a "userspace" program (a program above the kernel) can use to instruct the kernel to copy the contents of one file descriptor to another. You can use sendfile against a TCP socket to instruct the kernel to write a file's contents to a network connection. The advantage of this is that you don't ever need to read the file contents into a buffer in the Erlang program's memory. It's all up to the kernel to move those bytes around. Especially since Erlang is a garbage-collected language, a program can save some memory and work by using sendfile instead of reading and writing to a socket itself.

```erlang
upload(File) ->
    {ok, Socket} = gen_tcp:connect("my-unencrypted-host.example.com", 80, []),
    ok = gen_tcp:send(Socket, http1_header(File)),
    {ok, _BytesWritten} = file:sendfile(File, Socket),
    gen_tcp:close(Socket).
```

The rub is that `sendfile(2)` is a naive way to transfer bytes. It doesn't know about TLS encryption. Bytes are ferried from one file descriptor to the other (maybe a socket) as-is.

## kTLS

Enter kTLS: _kernel_ TLS. After performing a TLS handshake in userspace (your program) you can _offload_ the work of doing TLS encryption and decryption to the kernel by setting options on the socket with the connection's current TLS parameters. This makes things simpler in your userspace program: you don't need to encrypt anything, just send plain text on the socket. The kernel then transparently takes care of everything.

The really special thing about kTLS, for our purposes, is that it lets us treat a TLS encrypted network connection like a regular old unencrypted socket.

## kTLS + sendfile(2) uploads

The gist of the code using kTLS in Erlang would be:

```erlang
upload(File) ->
    %% Same as before: make a TLS handshake in userspace.
    {ok, SslSocket} = ssl:connect("s3.us-east-2.amazonaws.com", 443, [{ktls, true}]),
    %% Then handover TLS responsibilities to the kernel.
    Socket = do_ktls_handover(SslSocket),
    %% Now our socket is a regular old TCP socket! Unencrypted
    %% as far as we know!
    ok = gen_tcp:send(Socket, header(File)),
    {ok, _BytesWritten} = file:sendfile(File, Socket),
    gen_tcp:close(Socket).
```

I'm glossing over the details of the kTLS handover here intentionally. It's not well supported by Erlang yet and I even needed to patch Erlang to support the cipher currently negotiated by the S3 servers. For a full example see [this gist](https://gist.github.com/the-mikedavis/ceb1a246cdb03cb508bdf90382b6e162).

## Is it faster?

No! Actually I see the userspace and kernel versions of these uploads taking a very similar amount of time on the same size file (around 500 MiB). The optimization of `sendfile(2)` is not speed so much as memory. We avoid reading the file at all and instead let the kernel do it. So we need fancier system-wide benchmarks rather than a simple "upload time." In fact we can't even realize the full gains of this method without specialized hardware. (More below...)

But to quote Joe Armstrong in _Erlang and OTP in Action_:

> Make it work, then make it beautiful, then if you really, really have to, make it fast. 90 percent of the time, if you make it beautiful, it will already be fast. So really, just make it beautiful!

We've completely avoided TLS encryption and file reads in the userspace, and I think that's beautiful. I'm sure it's also performant in a loaded system, but I'll need to revisit this post with numbers when I can actually measure it.

In the meantime, we can talk about making it more beautiful.

## Hardware offload

The Linux kernel docs discuss a really juicy optimization: [zero-copy sendfile](https://docs.kernel.org/networking/tls.html#tls-tx-zerocopy-ro).

When kTLS is enabled, the encryption does not necessarily need to be done by the kernel itself. Instead, the kernel might pawn off job of encrypting data to the Network Interface Controller (NIC, also known as "network card"), if the NIC has the hardware-level support for doing the encryption itself. This means that instead of the encryption being done by your CPU on your RAM, the NIC takes care of everything. This lets the kernel treat the call to `sendfile(2)` as if it were a `sendfile(2)` call against a regular, unencrypted TCP socket. The kernel can send the bytes without any extra buffering for encryption.

The are some limitations to this approach: namely that the file can't be modified between the beginning and end of `sendfile(2)`. For cases where files are immutable once closed, though, as they might be when you upload them to S3, this is perfectly fine.

## Wrapping up

This was a quick look at using kTLS and `sendfile(2)` to upload files to S3 as efficiently as theoretically possible. I hope to revisit this post later with numbers and graphs showing how this impacts an actual system.

In the meantime here are a bunch of links for further reading:

* <https://erlangforums.com/t/introduce-kernel-tls-in-ssl-application/952>
* <https://github.com/erlang/otp/pull/6104>
* <https://delthas.fr/blog/2023/kernel-tls/>
* <https://netdevconf.org/1.2/papers/ktls.pdf>
* <https://netdevconf.info//0x14/pub/slides/25/TLS%20Perf%20Characterization%20slides%20-%20Netdev%200x14%20v2.pdf>
