---
layout: post
title: "Useful cuts from HTTP RFC"
author: Javier García
category: http
tags: http, protocol
---

This is just a collection of references I copy/pasted from the HTTP v1 RFC, mostly
for self-learning. I took specifically these snippets because they were the most
useful/insightful for me when reading it.

For the full RFC, link [here][0]. From the next section on, it's just cuts from
the RFC.

## Interesting remarks

Content negotiation: The mechanism for selecting the appropriate representation when servicing a request
The HTTP protocol is a request/response protocol

A more complicated situation occurs when one or more intermediaries are present
in the request/response chain. There are three common forms of intermediary:
proxy, gateway, and tunnel.

A gateway is a receiving agent, acting as a layer above some other server(s)
and, if necessary, translating the requests to the underlying server's
protocol.

HTTP communication usually takes place over TCP/IP connections. The default
port is TCP 80 , but other ports can be used. (...) HTTP only presumes a
reliable transport; any protocol that provides such guarantees can be used;

In HTTP/1.0, most implementations used a new connection for each
request/response exchange. In HTTP/1.1, a connection may be used for one or
more request/response exchanges

http_URL = "http:" "//" host [ ":" port ] [ abs_path [ "?" query ]]

If the port is empty or not given, port 80 is assumed.

If a proxy receives a host name which is not a fully qualified domain name, it
MAY add its domain to the host name it received. If a proxy receives a fully
qualified domain name, the proxy MUST NOT change the host name.

An empty abs_path is equivalent to an abs_path of "/".

Multiple message-header fields with the same field-name MAY be present in a
message if and only if the entire field-value for that header field is defined
as a comma-separated list [i.e., #(values)].

The transfer-length of a message is the length of the message-body as it
appears in the message; that is, after any transfer-codings have been applied.

For compatibility with HTTP/1.0 applications, HTTP/1.1 requests containing a
message-body MUST include a valid Content-Length header field unless the server
is known to be HTTP/1.1 compliant.

## "Syntax" TLDR

### Request

```
Request = Request-Line              ; Section 5.1
          *(( general-header        ; Section 4.5
           | request-header         ; Section 5.3
           | entity-header ) CRLF)  ; Section 7.1
          CRLF
          [ message-body ]          ; Section 4.3

Request-Line = Method SP Request-URI SP HTTP-Version CRLF
```

### Response

```
Response = Status-Line               ; Section 6.1
           *(( general-header        ; Section 4.5
            | response-header        ; Section 6.2
            | entity-header ) CRLF)  ; Section 7.1
           CRLF
           [ message-body ]          ; Section 7.2

Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

## On methods

The methods GET and HEAD MUST be supported by all general-purpose servers. All
other methods are OPTIONAL;

Response codes:

- 1xx: Informational - Request received, continuing process

- 2xx: Success - The action was successfully received,
  understood, and accepted

- 3xx: Redirection - Further action must be taken in order to
  complete the request

- 4xx: Client Error - The request contains bad syntax or cannot
  be fulfilled

- 5xx: Server Error - The server failed to fulfill an apparently
  valid request

HTTP status codes are extensible. HTTP applications are not required to
understand the meaning of all registered status codes, though such
understanding is obviously desirable. However, applications MUST understand the
class of any status code, as indicated by the first digit, and treat any
unrecognized response as being equivalent to the x00 status code of that class,
with the exception that an unrecognized response MUST NOT be cached.

### Codes:

```
Status-Code    =
      "100"  ; Section 10.1.1: Continue
    | "101"  ; Section 10.1.2: Switching Protocols
    | "200"  ; Section 10.2.1: OK
    | "201"  ; Section 10.2.2: Created
    | "202"  ; Section 10.2.3: Accepted
    | "203"  ; Section 10.2.4: Non-Authoritative Information
    | "204"  ; Section 10.2.5: No Content
    | "205"  ; Section 10.2.6: Reset Content
    | "206"  ; Section 10.2.7: Partial Content
    | "300"  ; Section 10.3.1: Multiple Choices
    | "301"  ; Section 10.3.2: Moved Permanently
    | "302"  ; Section 10.3.3: Found
    | "303"  ; Section 10.3.4: See Other
    | "304"  ; Section 10.3.5: Not Modified
    | "305"  ; Section 10.3.6: Use Proxy
    | "307"  ; Section 10.3.8: Temporary Redirect
    | "400"  ; Section 10.4.1: Bad Request
    | "401"  ; Section 10.4.2: Unauthorized
    | "402"  ; Section 10.4.3: Payment Required
    | "403"  ; Section 10.4.4: Forbidden
    | "404"  ; Section 10.4.5: Not Found
    | "405"  ; Section 10.4.6: Method Not Allowed
    | "406"  ; Section 10.4.7: Not Acceptable
    | "407"  ; Section 10.4.8: Proxy Authentication Required
    | "408"  ; Section 10.4.9: Request Time-out
    | "409"  ; Section 10.4.10: Conflict
    | "410"  ; Section 10.4.11: Gone
    | "411"  ; Section 10.4.12: Length Required
    | "412"  ; Section 10.4.13: Precondition Failed
    | "413"  ; Section 10.4.14: Request Entity Too Large
    | "414"  ; Section 10.4.15: Request-URI Too Large
    | "415"  ; Section 10.4.16: Unsupported Media Type
    | "416"  ; Section 10.4.17: Requested range not satisfiable
    | "417"  ; Section 10.4.18: Expectation Failed
    | "500"  ; Section 10.5.1: Internal Server Error
    | "501"  ; Section 10.5.2: Not Implemented
    | "502"  ; Section 10.5.3: Bad Gateway
    | "503"  ; Section 10.5.4: Service Unavailable
    | "504"  ; Section 10.5.5: Gateway Time-out
    | "505"  ; Section 10.5.6: HTTP Version not supported
    | extension-code
```

### Methods explained

The GET method means retrieve whatever information (in the form of an entity)
is identified by the Request-URI.

The POST method is used to request that the origin server accept the entity
enclosed in the request as a new subordinate of the resource identified by the
Request-URI in the Request-Line.

The PUT method requests that the enclosed entity be stored under the supplied
Request-URI. If the Request-URI refers to an already existing resource, the
enclosed entity SHOULD be considered as a modified version of the one residing
on the origin server.

This specification reserves the method name CONNECT for use with a proxy that
can dynamically switch to being a tunnel (e.g. SSL tunneling [44]).
If the media type remains unknown, the recipient SHOULD treat it as type
"application/octet-stream".

## About connections

Prior to persistent connections, a separate TCP connection was established to
fetch each URL, increasing the load on HTTP servers and causing congestion on
the Internet.

A significant difference between HTTP/1.1 and earlier versions of HTTP is that
persistent connections are the default behavior of any HTTP connection. That
is, unless otherwise indicated, the client SHOULD assume that the server will
maintain a persistent connection, even after error responses from the server.

Persistent connections provide a mechanism by which a client and a server can
signal the close of a TCP connection. This signaling takes place using the
Connection header field (section 14.10). Once a close has been signaled, the
client MUST NOT send any more requests on that connection.

Clients SHOULD NOT pipeline requests using non-idempotent methods or
non-idempotent sequences of methods (see section 9.1.2). Otherwise, a premature
termination of the transport connection could lead to indeterminate results.

(...) clients, servers, and proxies MUST be able to recover from asynchronous
close events. Client software SHOULD reopen the transport connection and
retransmit the aborted sequence of requests without user interaction so long as
the request sequence is idempotent (see section 9.1.2). Non-idempotent methods
or sequences MUST NOT be automatically retried, although user agents MAY offer
a human operator the choice of retrying the request(s).

The convention has been established that the GET and HEAD methods SHOULD NOT
have the significance of taking an action other than retrieval.

Methods can also have the property of "idempotence" in that (aside from error
or expiration issues) the side-effects of N > 0 identical requests is the same
as for a single request. The methods GET, HEAD, PUT and DELETE share this
property. Also, the methods OPTIONS and TRACE SHOULD NOT have side effects, and
so are inherently idempotent.

However, it is possible that a sequence of several requests is non- idempotent,
even if all of the methods executed in that sequence are idempotent.

By default, a response is cacheable if the requirements of the request method,
request header fields, and the response status indicate that it is cacheable.

[0]: https://tools.ietf.org/html/rfc2616
