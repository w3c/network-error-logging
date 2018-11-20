# Network Error Logging

Network Error Logging (NEL, pronounced "nell") allows server administrators to
instruct user agents to collect logs about requests to their server.  These logs
are roughly equivalent to server-side request logs, such as the [access.log][]
and [error.log][] files that you'd get from an Apache HTTP server.

[access.log]: https://httpd.apache.org/docs/2.4/logs.html#accesslog
[error.log]: https://httpd.apache.org/docs/2.4/logs.html#errorlog

## The problem

Server-side request logs provide useful information about every request that
makes it to your server.  These can be useful to diagnose problems that your end
users are encountering, especially when large groups of users are all
experiencing the same kind of problem.  You will typically pipe these logs into
a monitoring or observability pipeline of some kind, and either aggregate the
logs across interesting dimensions (end user country or network, server IP,
etc.), or keep the raw logs available and index them so that you can search for
individual requests.

However, there are interesting classes of network outages that **cannot** appear
in server-side logs: for instance, if the end user attempts to open a connection
to your server, but that connection attempt fails, then your server *by
definition* has not seen any request, and cannot include anything in server-side
logs to describe that request.

NEL solves this by collecting request logs on the client side, in the user
agent.  This ensures that you have visibility into network-level problems that
your end users are experiencing.

## Security and privacy aspects

There are two main security and privacy principles that Network Error Logging
adheres to:

  - **NEL reports are only visible to the server administrator.**

    You cannot use NEL to collect reports about servers or websites that you
    don't control.  NEL is configured separately for each origin, and is only
    enabled for secure connections.  Before using a NEL policy to create a
    report about a request, we verify that the policy was received from the same
    server that is handling that request.  (This helps prevent, for instance,
    using a DNS rebinding attack with NEL to collect information about a user's
    internal network.)

  - **NEL reports can only contain information that the server administrator
    already has access to.**

    You cannot use use to collect more information about your end users — if you
    can't see a particular piece of information at the server while processing
    the actual request, then we cannot include that information in a NEL report
    about the request.

## Configured via response headers

As the server administrator, you activate NEL for your origin by including
`Report-To` and `NEL` headers in your outgoing HTTP responses:

``` http
Report-To: { "group": "nel", "endpoints": [{"url": "https://example.com/reports"}], "max_age": 10886400 }
NEL: { "report_to": "nel", "max_age": 10886400 }
```

(Note that the `Report-To` header is defined by the [Reporting API][], which NEL
depends on for report delivery.  NEL itself only defines when reports about
network requests are created, and what format those reports have.)

The (required) `max_age` field specifies how long (in seconds) this report
collection policy will remain valid.  The (required) `report_to` field specifies
which [Reporting endpoint group][] these NEL reports will be delivered to.
There are several other optional NEL policy fields, described below.

[Reporting API]: https://w3c.github.io/reporting/
[Reporting endpoint group]: https://w3c.github.io/reporting/#concept-endpoint-groups

## Report contents

A NEL report contains a wealth of information about the request and response.
But to reiterate: it **does not** contain any information that isn't available
to server while it is processing the request.  (As just one example, a NEL
report does not include the IP address of the DNS resolver that the user agent
used to translate the request's domain name into an IP address.)

``` json
{
  "age": 10,
  "type": "network-error",
  "url": "https://www.example.com/",
  "body": {
    "sampling_fraction": 0.5,
    "referrer": "http://example.com/",
    "server_ip": "123.122.121.120",
    "protocol": "h2",
    "method": "GET",
    "request_headers": {},
    "response_headers": {},
    "status_code": 200,
    "elapsed_time": 823,
    "phase": "application",
    "type": "http.protocol.error"
  }
}
```

(Note that this spec only defines the contents of the `body` field; the other
top-level fields are part of the envelope defined by the [Reporting API][].)

## Sampling rates

For high-volume sites, you might not need to collect NEL reports about
**every** request that your servers receive.  NEL allows you to define
**sampling rates** that instruct user agents to only create reports about a
random subset of requests.  You can set separate sampling rates for
**successful** requests and **failed** requests.  Collecting reports about
successful requests lets you calculate error **rates**; that said, you usually
don't need much information about successful requests other than how many there
were, so you will often set the successful sampling rate relatively low (for
instance, 1-25%, depending on traffic volume).  On the other hand, failures are
much more interesting, and hopefully much rarer, so you will often set the
failure sampling rate very high.  (In fact, unless your website experiences a
**very** high rate of errors, it's usually fine to set your failure sampling
rate to 100%!)

You set sampling rates via the `success_fraction` and `failure_fraction` fields
of your NEL policy:

``` http
NEL: { ..., "success_fraction": 0.05, "failure_fraction": 0.5 }
```

The default `success_fraction` is 0.0; the default `failure_fraction` is 1.0.

## Reports about subdomains

Use can use NEL to collect information about DNS resolution failures for an
entire tree of domain names that you own, using the `include_subdomains` policy
field.  For instance, if you own `example.com`, you can provide the following
NEL policy:

``` http
NEL: { ..., "include_subdomains": true }
```

This policy would then apply to **all** subdomains of `example.com` that don't
provide a policy of their own.

However, owning the DNS tree rooted at `example.com` does **not** imply that
you also control all of the servers that any of those subdomains point at.
Therefore, a NEL report about a subdomain will **only** contain information
about the DNS resolution itself — that is, the part that you've been able to
assert ownership of.  The NEL report will not contain any information about
whether the connection to the server succeeded, or what response was received.

## Reflecting arbitrary request and response headers

You can have the user agent "reflect" the values of certain request or response
headers, using the `request_headers` and `response_headers` policy fields:

``` http
NEL: { ..., "request_headers": ["If-None-Match"], "response_headers": ["ETag"]}
```

This will cause the user agent to include copies of the `If-None-Match` request
header and `ETag` response header in NEL reports about this origin:

``` json
{
  "body": {
    "request_headers": {
      "If-None-Match": ["abcdef0123"]
    },
    "response_headers": {
      "ETag": ["abcdef0123"]
    }
  }
}
```

See [this section][cache validation example] of the spec for a more detailed
example.

[cache validation example]: https://w3c.github.io/network-error-logging/#monitoring-cache-validation
