# RPCPlugin Specification (draft)

> *NOTE*: This specification is not yet finalized and is still subject to
  change before final publication.

RPCPlugin is a set of conventions for applications to launch and communicate
with "plugin" programs that can extend application functionality at runtime.

Unlike some other plugin systems that are based around dynamic linking of
libraries into the calling program's address space, RPCPlugin launches plugins
as child processes and communicates with them over an RPC channel. That means
that the plugin code has its own execution context and memory address space.

Applications and plugins can be written in different languages, as long as
they both support the RPCPlugin protocol and agree on an application-level
RPC interface.

## Requirements and Notation

The words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY" and "OPTIONAL" in this
document are to be interpreted as described in [IETF BCP 14](IETF-BCP-14).

Each section of this document is _normative_ unless stated as _informative_.
Normative sections impose requirements for conformant implementations using
the keywords above. Informative sections do not impose any requirements and
instead provide advice to implementers.

This document describes syntax using the notation described in
[IETF RFC 5234: Augmented BNF for Syntax Specifications](IETF-RFC-5234),
including the extensions in
[IETF RFC 7405: Case-Sensitive String Support in ABNF](IETF-RFC-7405).

In addition to the core rules defined in RFC 5234 Appendix B, this
document assumes the following derived rules:

```abnf
integer = %x31-39 *(%x30-39) ; ASCII digits without leading zero
keyword = 1*(DIGIT / ALPHA)
base64  = *(DIGIT / ALPHA / "+" / "/") *2("=")
field   = *(%x21-7B / %x7D-7E) ; sequence of all printable
                               ; ASCII characters except "|"
```

[IETF-BCP-14]: https://tools.ietf.org/html/bcp14
[IETF-RFC-5234]: https://tools.ietf.org/html/rfc5234
[IETF-RFC-7405]: https://tools.ietf.org/html/rfc7405

## System-defined Behaviors

The protocol described in this document includes interactions with features of
a host operating system to allow a calling application to launch and
communicate with a separate process containing plugin code.

The exact details of these mechanisms varies between operating systems. This
document requires a host operating system that is able to provide for the
following:

* Ability for one running program (the _client_) to launch a second program
  (the _server_) with a separate memory address space.

* Ability for the client to pass a number of key/value pairs where keys and
  values are both strings to the server while launching it. This document
  will refer to this mechanism as _environment variables_, matching the
  common name for this feature in current operating systems.

* Ability for the server to transmit a sequence of bytes to its calling client
  once it has launched and initialized. This document will refer to this
  mechanism as the server's _standard output stream_, matching the common name
  for this feature in current operating systems.

* Ability for the server to listen on a network socket that is reachable from
  the client process and initialize stream sockets for primary RPC
  communication. Each operating system binding will define one or more
  _transport protocols_ that bind to particular socket-like communication
  primitives available on that operating system, such as TCP or Unix domain
  sockets.

* Ability for the server to signal the client to terminate. This document
  will refer to this mechanism as the _kill signal_, matching the common name
  for this feature in current operating systems.

Later sections of this document define the mapping of the above to operating
system primitives for Unix-like (POSIX) operating systems and for Microsoft
Windows. Other specifications may define the corresponding mapping to
primitives from other operating systems.

## Application-defined Behaviors

This document uses the term "application-defined" to indicate when an
application designer must make a decision and document that decision as part
of the application-level protocol definition.

This document may constrain the parameters of that decision or offer
informative advice to aid in the decision, but the final choice is made by
the application designer and not defined by this document.

## Protocol Roles

The RPCPlugin protocol specifies interactions between two different parties:

- The _server_ is the plugin itself, which exposes an RPC interface to its
  calling application.
- The _client_ is the host application, which launches one or more plugin
  servers.

The client locates an executable program that will provide the server, using
an application-defined mechanism, and launches that program with a set of
environment variables that the server program can use to determine the
capabilities of the client. The server program then opens a listen socket
on which it will accept RPC requests and writes information about it to
its standard output stream so that the client can connect.

RPCPlugin also defines an optional TLS auto-negotiation protocol wherein the
client and server can generate and exchange temporary TLS certificates for 
mutual authentication.

## Application-level Protocol Definition

RPCPlugin defines a mechanism for launching a plugin and locating its RPC
server. The particular set of RPC messages available on the resulting
server are application-defined.

Application-specific RPC protocols are defined in terms of [gRPC](gRPC).
Typically the protocol is defined in
[the Protocol Buffers schema language](protobuf-schema) and can use any
features supported by gRPC.

RPCPlugin includes a version negotiation protocol to allow clients and servers
to potentially support multiple protocol versions and automatically select one
that both parties support. Application protocol designers SHOULD assign each
protocol version an integer major version number, and ensure that each major
version uses a different Protocol Buffers _package_ name.

Application protocol designers MAY also use minor version numbering to indicate
successive protocol versions that are backwards-compatible. The RPCPlugin
negotiation protocol does not consider minor versions, so negotiation of
minor version enhancements must be defined at the application layer.

[gRPC]: https://grpc.io/
[protobuf-schema]: https://developers.google.com/protocol-buffers/docs/proto3

## Plugin Initialization and Handshake Protocol

### Launching the Server Program

After locating the executable program for a plugin server in an
application-defined manner, the client requests the operating system to launch
that program.

The server program MUST be launched with the following environment variables:

* `PLUGIN_PROTOCOL_VERSIONS`: comma-separated sequence of integer
  application protocol major versions supported by the client encoded in
  decimal notation using ASCII digits.

* `PLUGIN_TRANSPORTS`: comma-separated sequences of keywords representing
  operating-system-specific RPC transport protocols supported by the client.

* `PLUGIN_CLIENT_CERT`: when TLS auto-negotation is used, a
  [PEM-encoded](PEM-encoding)
  TLS certificate that the client will use to authenticate itself when
  connecting to the RPC server.

The `PLUGIN_PROTOCOL_VERSIONS` and `PLUGIN_TRANSPORTS` values conform to
the following grammars, respectively:

```abnf
protocol-versions-spec = integer *("," integer)
transports-spec        = keyword *("," keyword)
```

In addition to the above environment variables, each application must define
one application-defined _cookie environment variable_ name and constant
value that the client will set and the server will check as a heuristic for
whether it appears to be running as a plugin server. The client MUST set this
environment variable when launching the server, and the server MUST check that
it is set and exit immediately with an error if not.

The presence and meaning of any other environment variables is
application-defined.

The server program MUST be launched with its standard output channel configured
such that data written to it will be recieved by the client program.
The use of any other input and output streams, including a "standard error"
stream on operating systems with such a concept, is application-defined.

If the operating system indicates that it was unable to launch the server
program or if the server program exits before producing its handshake
response (as described in the following section), plugin initialization fails.

[PEM-encoding]: https://tools.ietf.org/html/rfc7468#section-5

### Server Handshake Response

Once started, the server program must inspect the `PLUGIN_PROTOCOL_VERSIONS`
and `PLUGIN_TRANSPORTS` environment variables and compare the indicated
appication protocol and RPC transports with those the server is able to
support.

The server must select the greatest protocol version number it has in common
with the client. This is the _negotiated protocol version_. If the client and
server have no protocol versions in common, the server program MUST exit
immediately, and plugin initialization fails.

The server must select a single transport protocol that it has in common with
the client. The available transports are system-defined, and so the
system-specific mapping may define a preference order for its transports. If
no system-specific preference order applies, the preference order is
application-defined. This is the _negotiated RPC transport_. If the client and
server have no supported transports in common, the server program MUST exit
immediately, and plugin initialization fails.

After selecting a negotiated protocol version and a negotiated RPC transport,
the server MUST create an RPC listener at a transport-specific address,
offering a [gRPC](gRPC) server that supports the application-defined service
for the negotiated protocol version.

The client completes negotiation by producing a negotiation response. The
negotiation response is a sequence of bytes written to its standard output
stream which conforms to the following grammar:

```abnf
handshake-response = (
    "1"            ; RPCPlugin handshake version
    "|" integer    ; negotiated protocol version
    "|" keyword    ; negotiated RPC transport
    "|" field      ; transport-specific endpoint specification
    [ "|" base64 ] ; temporary server TLS certificate
    LF
)
```

The negotiated protocol version and the negotiated RPC transport inform the
client of the selections made by the server. These MUST each match one of
the options advertised by the client in the negotiation environment variables.
If not, plugin initialization fails and the client should send the server
process the kill signal.

The transport-specific endpoint specification is a sequence of bytes whose
interpretation is defined by the negotiated RPC transport that contain
sufficient information for the client to connect and send RPC messages to
the endpoint.

After the client receives and accepts the handshake response, plugin
initialization is successful. The client may now send [gRPC](gRPC) messages
to the given endpoint that conform to the application-specific protocol.

## Shutting Down a Plugin Server

The client MUST shut down all of the server programs it has started before
the client terminates. The client MAY shut down server programs earlier, if
the plugin is no longer required per application-defined behavior.

A client shuts down a plugin server by sending it a kill signal. The server
is immediately terminated and gets no opportunity to take any cleanup actions.
A client MUST NOT shut down a plugin server while an RPC request is in
progress.

Application protocol designers MUST design protocol actions such that each
operation is self-contained in terms of required clean-up actions, so that
it is always safe to terminate the server program at any instant when there
is no RPC request in progress.

## Temporary TLS Certificate Negotiation

RPCPlugin requires client and server implementations to use TLS 1.2 or later
for the negotiated RPC channel. Applications MAY use the mechanism described
in this section to allow clients and servers to generate temporary private
keys and certificates and share the certificates in order to allow
authenticated communication between a client and a server that have no prior
trust relationship.

An application MAY define other out-of-band mechanisms for certificate
negotiation, and not use the mechanism described in this section. Client
implementations for such an application MUST NOT set `PLUGIN_CLIENT_CERT`
when launching a server program, and server implementations for such an
application MUST omit the server certificate portion of the handshake
response message.

If an application defines that its plugin protocol uses automatic temporary
TLS certificate negotiation, both client and server MUST meet the requirements
in this section.

### Temporary Keys and Certificates

Both client and server MUST generate temporary keys and certificates when
using the automatic temporary TLS certificate negotiation mechanism. In
both cases, the key and certificate are generated per the following
requirements.

The server MUST generate is key using a random number source with sufficient
entropy and a suitable algorithm to produce a private key for TLS
communication. This document does not prescribe any particular key algorithm
or key size, but implementors should review current best practices for
key algorithm and compatibility across deployed TLS implementations and
select a suitable key type to meet an application's security and compatibility
needs.

The server must then generate a self-signed [X.509](X509) certificate meeting
the following requirements:

* The subject's common name MUST be `localhost`.
* The subject DNS name MUST be `localhost`.
* The key usage field MUST include the following usages:
  * Digital signature
  * Key Encipherment
  * Key Agreement
  * Certificate Signing
* The extended key usage extension MUST be included, and indicate that
  both client authentication and server authentication are permitted
  usages.
* The Certificate Authority flag MUST be set.
* The serial MUST be a pseudorandom number.
* The "not before" timestamp SHOULD be set to 30 seconds before the time
  when the certificate is being generated.
* The "not after" timestamp SHOULD be set to ensure that the certificate
  will not expire before the client kills the server program.
* The certificate MUST be signed with the temporary private key.
* The certificate MUST describe the public key associated with the temporary
  private key.

[X509]: https://www.itu.int/rec/T-REC-X.509-201610-I/en

### Temporary Client Certificate

Before launching a server program, the client MUST generate a suitable
private key and certificate that it will use for communication
with that server. A server MAY generate a single key and secret to use with
many server programs, or it MAY generate one key and secret per server
program.

The client MUST produce a serialization of the generated certificate using
[PEM encoding](PEM-encoding) and include the result as the value of the
`PLUGIN_CLIENT_CERT` environment variable when launching the server program.

### Temporary Server Certificate

After selecting the negotiated protocol version and the negotiated RPC
transport, the server MUST generate a suitable private key and certificate
that it will use in communication with the client. The server MUST generate
a unique key and secret on each execution.

The server MUST produce a DER-encoded ASN.1 certificate structure and then
encode the result using [base64 encoding](Base64), including any necessary
padding suffix characters.

The encoded certificate data MUST be included as the final field of the
handshake response.
Full PEM encoding MUST NOT be applied to the temporary server certificate
before inclusion in the handshake response.

[Base64]: https://tools.ietf.org/html/rfc4648#section-4

### Certificate Authentication

After initialization is complete, both client and server know their own keys
and certificates and they each know the certificate of the other party.

The client MUST configure its gRPC client to present its own certificate for
authentication during a TLS handshake. It MUST configure the gRPC client
to accept _only_ the server's certificate as trusted. The client MUST
disable any general root certificate store provided by the host operating
system or the TLS implementation.

The server MUST configure its gRPC listener to present its own certificate
for authentication during a TLS handshake. It MUST configure the gRPC listener
to require client authentication and accept _only_ the client's certificate.
The client MUST disable acceptance any other certificates.

Because RPCPlugin RPC communication is over system-local channels rather than
networks, temporary certificates have their subject set to the DNS name
`localhost`. The DNS name is not significant for RPCPlugin authentication
and RPCPlugin implementations MAY disregard the certificate subject and
MAY treat `localhost` as an acceptable hostname regardless of the details
of the negotiated transport endpoint.

Applications using out-of-band certificate exchange rather than temporary
TLS negotation are not subject to the requirements in this section. The
application protocol specification MUST specify a corresponding set of
TLS configuration requirements that are appropriate for the application's
certificate distribution mechanism and usage pattern.

## RPC Transports

This document defines two initial RPC transports, described in the following
sections. Other specifications may define other transports.

[GRPC-over-HTTP2]: https://github.com/grpc/grpc/blob/v1.23.0/doc/PROTOCOL-HTTP2.md

### TCP Socket Transport

The TCP socket transport transmits RPC messages via a TCP server. It is
identified by the keyword "tcp".

When the negotiated RPC transport is the TCP socket transport, the server
listens on a randomly-selected TCP port on a system-defined interface. The
endpoint specification conforms to the following grammar:

```abnf
tcp-endpoint = tcp-host ":" tcp-port
tcp-host     = field
tcp-port     = integer
```

The `tcp-host` portion is an IP address or DNS name for the interface where
the TCP server is listening. The `tcp-port` is the port number of the server,
given in decimal using ASCII digits.

The host portion must use standard IPv4 address, IPv6 address, or DNS name
notation. If an IPv6 address is used, it must be given in brackets `[` `]`
to avoid ambiguity between the colon separators in the address and the
colon introducing the `tcp-port` portion.

Communication over the TCP socket is per [gRPC over HTTP2](GRPC-over-HTTP2),
using the `application/grpc+proto` Content-Type. TLS 1.2 or higher is
mandatory. If the application uses TLS auto-negotiation then the client
must present its temporary certificate and verify the server's certificate,
and the server must present its temporary certificate and verify the client's.

The TCP Socket Transport is primarily intended for communication via a
system's loopback interface. The operating system-specific bindings in this
specification each describe how the TCP Socket Transport is to be used on
its target platform. Other specifications MAY define mechanisms for using
the TCP Socket Transport over a local-area or wide-area network, but that is
outside the scope of this specification.

### Unix Domain Socket Transport

The Unix domain socket transport transmits RPC messages via a stream-oriented
Unix domain socket server. It is identified by the keyword "unix".

When the negotiated RPC transport is the Unix domain socket transport, the
server establishes a listen socket at a location that is also accessible to
the client. The endpoint specification is an absolute path to the socket
using system-defined path syntax.

Communication over the Unix socket is per [gRPC over HTTP2](GRPC-over-HTTP2),
using the `application/grpc+proto` Content-Type. TLS 1.2 or higher is
mandatory. If the application uses TLS auto-negotiation then the client
must present its temporary certificate and verify the server's certificate,
and the server must present its temporary certificate and verify the client's.

## System-specific Definitions

RPCPlugin is intended to be portable to various operating systems, but the need
to launch and communicate with separate programs requires interacting with
operating system-specific features that have different details depending on
the target platform.

The following sections define a binding of the RPCPlugin handshake process
to Unix-like (POSIX) operating systems and Microsoft Windows systems
respectively. Bindings for other operating systems may be defined in other
specifications.

### Binding for Unix-like (POSIX) operating systems

This section defines a binding of the RPCPlugin handshake process to platform
features defined by the POSIX specifications. This binding is applicable both
to POSIX-certified operating systems and to systems that offer a compatible
subset of the POSIX APIs sufficient to implement the described binding.

The following requirements mention specific POSIX API functions and features.
A specific implementation MAY use other POSIX API functions instead of the
ones indicated, as long as the effect (as visible to the other party) is
identical to that specified for the given function. A specific implementation
MAY use an operating system-specific system call interface directly, bypassing
POSIX API functions, as long as the effect (as visible to the other party) is
identical to that specified for the given POSIX function.

A client MUST launch a selected server program via the following procedure:

* Call `pipe` to establish a pipe that will serve as the handshake response
  channel.

* Call `fork` to create a child process. The remaining steps occur only
  in the child process.

* Use `dup2` to associate the `PIPE_WRITE` descriptor returned from `pipe`
  above with the file descriptor `STDOUT_FILENO`.

* Call `execle` or `execpe` to load the selected server program and begin
  executing it in the child process. Set the `envp` argument to include
  the environment variables as described in earlier sections, using the
  specified names exactly as given, case-sensitive.

* The server program must then call `getenv` to retrieve each of the
  environment variables in order to implement the handshake procedure
  described in earlier sections.

The client must then await the handshake response message on the `PIPE_READ`
file descriptor returned from the `pipe` call.

Both the TCP Socket Transport and the Unix Domain Socket Transport are
permitted on Unix-like operating systems. Implementations SHOULD support both
transports if available on the target operating system.

If client and server both support the TCP Socket Transport and the Unix Domain
Socket Transport, the client must give preference to the Unix Domain Socket
Transport during negotiation.

When the negotiated RPC transport is the TCP Socket Transport, the server
MUST pseudo-randomly select an unused TCP port on the interface holding the
IPv4 address `127.0.0.1` and bind a listen socket to that port before
indicating the resulting endpoint in the handshake response.

When the negotiated RPC transport is the Unix Domain Socket Transport, the
server MUST establish a Unix domain socket in a temporary directory accessible
to both the server and the client and listen on that socket before indicating
the absolute path to that socket as the endpoint in the handshake response.

The client SHOULD implement the [XDG Base Directory Specification](XDG-Basedir)
and create its socket using an unpredictable path under `XDG_RUNTIME_DIR`
if possible. The client MAY instead use `mkdtemp` to establish a temporary 
directory into which a socket can be written.

Both the TCP Socket Transport and the Unix Domain Socket Transport assume that
the client and server processes have a common view of either the network
interface holding the address `127.0.0.1` or the filesystem location containing
the Unix domain socket respectively. On platforms that support per-process
network or filesystem namespaces, the client and server MUST use the same
namespaces.

The client MUST construct a handshake response message and write it to
its stdout. If writing to stdout via a buffered stream, the client MUST
flush that buffer immediately after the response message is written in full
in order to avoid a deadlock situation.

When a client must send a kill signal to a server, it must call `kill` with
the process id (pid) of the server process and the signal `SIGKILL`.

[XDG-Basedir]: https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html

### Binding for Microsoft Windows operating systems

This section defines a binding of the RPCPlugin handshake process to platform
features from the "Win32" API.

The following requirements refer to specific Win32 API functions. A specific
implementation MAY use other Win32 API features instead, as long as the
result is equivalent (from the perspective of the other party) to the
functions indicated.

A client MUST launch a selected server program via the following procedure:

* Call `CreatePipe` to establish an anonymous pipe that will serve as the
  handshake response channel.

* Call `SetHandleInformation` to modify the "read" handle of the pipe to set
  `HANDLE_FLAG_INHERIT` to `0`, ensuring that the server program will not
  inherit it.

* Call `CreateProcess` to launch the server program:
  * Set the command line to refer to the selected server program.
  * In the `STARTUPINFO` value, set `hStdOutput` to the "write" handle of
    the pipe, and ensure that the `STARTF_USESTDHANDLES` bit is set in
    `dwFlags`.
  * Set `lpEnvironment` to include the environment variables required by
    earlier sections. Environment variable names are conventionally
    case-insensitive on Windows, but the client SHOULD set the names defined
    by this document in uppercase, exactly as indicated.

* Retain the process handle for the child process and the "read" handle from
  the pipe for later operations.

After the launching the server program, the client awaits its handshake
response using `ReadFile` with the "read" handle from the communication pipe.

The TCP Socket Transport is available on Windows. The Unix Domain Socket
Transport is not defined for Windows and SHOULD NOT be used, even though
Unix Domain Socket functionality is available on some Windows versions.

When the negotiated RPC transport is the TCP Socket Transport, the server
MUST pseudo-randomly select an unused TCP port on the interface holding the
IPv4 address `127.0.0.1` and bind a listen socket to that port before
indicating the resulting endpoint in the handshake response.

Both TCP Socket Transport assumes that the client and server processes have a
common view of either the network interface holding the address `127.0.0.1`
and that no firewall software will block communication between local
processes. This document defines no mechanism to ensure successful
connectivity.

The client program MUST call `GetStdHandle(STD_OUTPUT_HANDLE)` to obtain the
standard output handle, and then construct a handshake response message and
write it to that handle using `WriteFile`. If writing to the standard output
handle via a buffered stream, the client MUST flush that buffer immediately
after the response message is written in full in order to avoid a deadlock
situation.

When a client must send a kill signal to a server, it must call
`TerminateProcess` with the process handle of the server process and a
non-zero exit code.

## Security Considerations

The application-defined RPC protocol may include messages that transmit
sensitive information from client to server or server to client.
The use of TLS for the RPC protocol achieves both authentication and
confidentality for that communication channel as long as both parties are
able to protect their private keys from reading by other parties.

Client and server implementations SHOULD retain temporary private keys
generated during certificate auto-negotiation in memory only, relying on
memory protection features of the operating system and hardware.
Some platforms may be vulnerable to side-channel attacks that permit a
malicious program to bypass memory protection; process memory isolation
is commonly a requirement for effective mitigation of such side-channel
attack vectors.

Certificate auto-negotiation relies on exchange of certificate data via
both environment variables (from client to server) and operating system
pipe primitives (from server to client). Any software that is able to insert
itself as an intermediary or is able to modify the server process environment
early in its runtime may be able to substitute its own certificate(s) and
intercept or modify client/server communication via the RPC channel.

Certificate auto-negotiation requires both client and server to have access
to sufficient entropy to generate a strong random private key. Systems
that were booted only recently may not have sufficient entropy early in
the boot process; for applications that start during system boot and
immediately launch plugins, consider whether the target system will have
sufficient entropy for temporary key generation at that time, and either
arrange to provide suitable entropy or design the application to use
out-of-band TLS certificate exchange with predefined keys and certificates.

Applications that elect to use out-of-band TLS certificate exchange mechanisms
assume responsibility for secure generation and exchange of those certificates.
RPCPlugin's authentication model assumes that each party has its own key
and that no other process is able to obtain that key.

## Appendices

### HashiCorp "go-plugin" Compatibility

> This section is *informative.*

The RPCPlugin handshake protocol is based on the handshake protocol for a
similar system "go-plugin" designed and implemented by HashiCorp.

RPCPlugin is not _strictly_ a subset of go-plugin, but it is close enough to
a subset that it is possible to create an RPCPlugin implementation that can
interoperate with go-plugin under certain constraints.

This section includes non-normative advice on adjustments that can be made
for client and server implementations of RPCPlugin in order to achieve
interoperability with existing go-plugin implementations of the other party.
go-plugin compatibility is not required by this document, but may be
required in the protocol definition for a specific application. Such a
specification should provide normative requirements for achieving that
compatibility, possibly guided by the advice in the following sub-sections.

#### General Differences

HashiCorp go-plugin was initially based on an RPC protocol specific to the
Go programming language, known in the Go ecosystem as "net/rpc". It was
later retrofitted with support for gRPC. RPCPlugin is not compatible with
the `net/rpc` form of go-plugin, but it _can_ be compatible with certain
applications using go-plugin with gRPC.

The go-plugin protocol is emergent from its implementation rather than
formally specified. This section and the following sections describe go-plugin
as of its library release v1.0.0. Later versions of go-plugin may have
different behaviors that diverge further from the RPCPlugin protocol.

#### RPCPlugin Clients with go-plugin Servers

go-plugin servers do not respect the `PLUGIN_TRANSPORTS` environment
variable. The primary Go implementation of go-plugin always selects the
Unix Domain Socket transport on Unix systems and the TCP Socket Transport
on Windows systems. An RPCPlugin client that must work with go-plugin
servers on Unix systems _must_ support the Unix Domain Socket transport.

go-plugin servers use a slightly different format for the temporary
server TLS certificate in the handshake response. RPCPlugin requires standard
base64 encoding with padding, but go-plugin does not produce padding
characters. An RPCPlugin client that must work with go-plugin servers
must be tolerant of un-padded base64 encoding of temporary server TLS
certificates.

#### go-plugin Clients with RPCPlugin Servers

go-plugin clients do not set `PLUGIN_TRANSPORTS` when launching server
programs. The primary Go implementation of `go-plugin` supports both
the TCP Socket Transport and the Unix Domain Socket transport when running
on a Unix system, and only the TCP SOcket Transport when running on a
Windows system. An RPCPlugin server that must work with go-plugin clients
must assume default values when `PLUGIN_TRANSPORTS` is not set:
"unix,tcp" on Unix systems and "tcp" on Windows systems.

go-plugin clients expect a slightly different format for the temporary
server TLS certificate in the server's handshake response. RPCPlugin requires
clients to produce standard base64 encoding with padding, but go-plugin
expects no padding characters. An RPCPlugin server that must work with
go-plugin clients must ensure that its generated certificate is of a length
that can encode to standard base64 without the need for padding characters,
such as by adding padding to a text field in the certificate whose meaning
is not significant for RPCPlugin authentication.

### Detaching and Re-attaching Clients

> This section is *informative.*

The RPCPlugin handshake protocol is defined under the assumption that a
client will always outlive the servers it launched.

For certain application types, it may be desireable to relaunch a client
process without killing and re-launching existing plugin servers. This
capability is not part of the RPCPlugin protocol definition, but this section
contains some advise for application designers who intend to implement
such a mechanism as an extension of the protocol in this document.

An initial constraint on such detach and re-attach behavior is ensuring that
the client process can exit without causing the operating system to kill
the running servers. On some platforms, that may require the client to
"daemonize" the server process as part of launching it, so that it belongs
to a separate session from the client and can be controlled independently.
Alternatively, the application could be designed such that the RPCPlugin
client and servers are both started as child processes of a supervisor
process, with the supervisor implementing the initialization and negotiation
protocol before sending the handshake results to the client process.

Re-attaching a new client to a running server requires retaining all of the
state that the original client determined during the initialization and
negotiation protocol:

* The negotiated protocol version
* The negotiated RPC transport
* The RPC endpoint information
* Any temporary private key and certificates determined using the
  automatic negotiation mechanism

The temporary private key in particular must be handled with care to ensure
that retaining it does not make it available to malicious other processes
on the system that may try to impersonate the intended client when interacting
with the running server. Applications that need client re-attaching behavior
may be better served by using out-of-band certificate exchange mechanisms.

In some cases it will be possible that the server is no longer running at the
time the new client attempts to re-attach. For example, the server may have
crashed and had no supervisor process ready to relaunch it. In that case,
the application must include a mechanism to initialize and negotiate with a
new instance of the server program, discarding any retained negotiation
information from the previous server.
