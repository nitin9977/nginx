# nginx — Introduction & Configuration Guides

**Source**: https://nginx.org/en/docs/  

Summaries of the introduction and guide pages from the official nginx documentation.

---

## Config file measurement units

**Source**: https://nginx.org/en/docs/syntax.html  

### Configuration file measurement units
Sizes and offsetsTime intervals
nginx supports several measurement units for specifying sizes, offsets, and time intervals within configuration files.

#### Sizes and offsets

Sizes can be specified in bytes, kilobytes, or megabytes using the following suffixes:
 

- k and K for kilobytes

- m and M for megabytes

For example, “ 1024 ”, “ 8k ”, “ 1m ”.

Offsets can be also specified in gigabytes using the g or G suffixes.

#### Time intervals

Time intervals can be specified in milliseconds, seconds, minutes, hours, days and so on, using the following suffixes: ms milliseconds s seconds (default) m minutes h hours d days w weeks M months, 30 days y years, 365 days

Multiple units can be combined in a single value by specifying them in the order from the most to the least significant, and optionally separated by whitespace. For example, “ 1h 30m ” specifies the same time as “ 90m ” or “ 5400s ”.

 

- A value without a suffix means seconds.

- It is recommended to always specify a suffix.

- Certain time intervals can be specified only with a seconds resolution.

---

## Connection processing methods

**Source**: https://nginx.org/en/docs/events.html  

### Connection processing methods

nginx supports a variety of connection processing methods. The availability of a particular method depends on the platform used. On platforms that support several methods nginx will normally select the most efficient method automatically. However, if needed, a connection processing method can be selected explicitly with the use directive.

The following connection processing methods are supported:
 

- select — standard method. The supporting module is built automatically on platforms that lack more efficient methods. The --with-select_module and --without-select_module configuration parameters can be used to forcibly enable or disable the build of this module.

- poll — standard method. The supporting module is built automatically on platforms that lack more efficient methods. The --with-poll_module and --without-poll_module configuration parameters can be used to forcibly enable or disable the build of this module.

- kqueue — efficient method used on FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0, and macOS.

- epoll — efficient method used on Linux 2.6+. The EPOLLRDHUP (Linux 2.6.17, glibc 2.8) and EPOLLEXCLUSIVE (Linux 4.5, glibc 2.24) flags are supported since 1.11.3. Some older distributions like SuSE 8.2 provide patches that add epoll support to 2.4 kernels.

- /dev/poll — efficient method used on Solaris 7 11/99+, HP/UX 11.22+ (eventport), IRIX 6.5.15+, and Tru64 UNIX 5.1A+.

- eventport — event ports, method used on Solaris 10+ (due to known issues, it is recommended using the /dev/poll method instead).

---

## Setting up hashes

**Source**: https://nginx.org/en/docs/hash.html  

### Setting up hashes

To quickly process static sets of data such as server names, map directive’s values, MIME types, names of request header strings, nginx uses hash tables. During the start and each re-configuration nginx selects the minimum possible sizes of hash tables such that the bucket size that stores keys with identical hash values does not exceed the configured parameter (hash bucket size). The size of a table is expressed in buckets. The adjustment is continued until the table size exceeds the hash max size parameter. Most hashes have the corresponding directives that allow changing these parameters, for example, for the server names hash they are server_names_hash_max_size and server_names_hash_bucket_size .

The hash bucket size parameter is aligned to the size that is a multiple of the processor’s cache line size. This speeds up key search in a hash on modern processors by reducing the number of memory accesses. If hash bucket size is equal to one processor’s cache line size then the number of memory accesses during the key search will be two in the worst case — first to compute the bucket address, and second during the key search inside the bucket. Therefore, if nginx emits the message requesting to increase either hash max size or hash bucket size then the first parameter should first be increased.

---

## Logging to syslog

**Source**: https://nginx.org/en/docs/syslog.html  

### Logging to syslog

The error_log and access_log directives support logging to syslog. The following parameters configure logging to syslog:
 

`server=``address`

Defines the address of a syslog server.
The address can be specified as a domain name or IP address,
with an optional port, or as a UNIX-domain socket path
specified after the “`unix:`” prefix.
If port is not specified, the UDP port 514 is used.
If a domain name resolves to several IP addresses, the first resolved
address is used.

`facility=``string`

Sets facility of syslog messages, as defined in
RFC 3164.
Facility can be one of “`kern`”, “`user`”,
“`mail`”, “`daemon`”,
“`auth`”, “`intern`”,
“`lpr`”, “`news`”, “`uucp`”,
“`clock`”, “`authpriv`”,
“`ftp`”, “`ntp`”, “`audit`”,
“`alert`”, “`cron`”,
“`local0`”..“`local7`”.
Default is “`local7`”.

`severity=``string`

Sets severity of syslog messages for
access_log,
as defined in
RFC 3164.
Possible values are the same as for the second parameter (level) of the
error_log directive.
Default is “`info`”.

Severity of error messages is determined by nginx, thus the parameter
is ignored in the `error_log` directive.

`tag=``string`

Sets the tag of syslog messages.
Default is “`nginx`”.

`nohostname`

Disables adding the “hostname” field into the syslog message header (1.9.7).

Example syslog configuration:
 
```nginx

error_log syslog:server=192.168.1.1 debug;

access_log syslog:server=unix:/var/log/nginx.sock,nohostname;
access_log syslog:server=[2001:db8::1]:12345,facility=local7,tag=nginx,severity=info combined;

```

 
Logging to syslog is available since version 1.7.1.
As part of our
commercial subscription
logging to syslog is available since version 1.5.3.

---

## Debugging log

**Source**: https://nginx.org/en/docs/debugging_log.html  

### A debugging log
Debugging log for selected clientsLogging to a cyclic memory buffer
To enable a debugging log, nginx needs to be configured to support debugging during the build:
 
```nginx

./configure --with-debug ...

```

Then the debug level should be set with the error_log directive:
 
```nginx

error_log /path/to/log debug;

```

To verify that nginx is configured to support debugging, run the nginx -V command:
 
```nginx

configure arguments: --with-debug ...

```

Pre-built Linux packages provide out-of-the-box support for debugging log with the nginx-debug binary (1.9.8) which can be run using commands
 
```nginx

service nginx stop
service nginx-debug start

```

and then set the debug level. The nginx binary version for Windows is always built with the debugging log support, so only setting the debug level will suffice.

Note that redefining the log without also specifying the debug level will disable the debugging log. In the example below, redefining the log on the server level disables the debugging log for this server:
 
```nginx

error_log /path/to/log debug;

http {
    server {
        error_log /path/to/log;
        ...

```

To avoid this, either the line redefining the log should be commented out, or the debug level specification should also be added:
 
```nginx

error_log /path/to/log debug;

http {
    server {
        error_log /path/to/log debug;
        ...

```

#### Debugging log for selected clients

It is also possible to enable the debugging log for selected client addresses only:
 
```nginx

error_log /path/to/log;

events {
    debug_connection 192.168.1.1;
    debug_connection 192.168.10.0/24;
}

```

#### Logging to a cyclic memory buffer

The debugging log can be written to a cyclic memory buffer:
 
```nginx

error_log memory:32m debug;

```

Logging to the memory buffer on the debug level does not have significant impact on performance even under high load. In this case, the log can be extracted using a gdb script like the following one:
 
```nginx

set $log = ngx_cycle->log

while $log->writer != ngx_log_memory_writer
    set $log = $log->next
end

set $buf = (ngx_log_memory_buf_t *) $log->wdata
dump binary memory debug_log.txt $buf->start $buf->end

```

Or using an lldb script as follows:
 
```nginx

expr ngx_log_t *$log = ngx_cycle->log
expr while ($log->writer != ngx_log_memory_writer) { $log = $log->next; }
expr ngx_log_memory_buf_t *$buf = (ngx_log_memory_buf_t *) $log->wdata
memory read --force --outfile debug_log.txt --binary $buf->start $buf->end

```

---

## Command-line parameters

**Source**: https://nginx.org/en/docs/switches.html  

### Command-line parameters

nginx supports the following command-line parameters:
 

- -? | -h — print help for command-line parameters.

- -c file — use an alternative configuration file instead of a default file.

- -e file — use an alternative error log file to store the log instead of a default file (1.19.5). The special value stderr selects the standard error file.

- -g directives — set global configuration directives , for example, ```nginx nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw.ncpu`;" ```

- -l port — enable nginx control REST API on a specified port or UNIX-domain socket (1.29.8). This parameter is available as part of our commercial subscription .

- -p prefix — set nginx path prefix, i.e. a directory that will keep server files (default value is /usr/local/nginx ).

- -q — suppress non-error messages during configuration testing.

- -s signal — send a signal to the master process. The argument signal can be one of: stop — shut down quickly

- quit — shut down gracefully

- reload — reload configuration, start the new worker process with a new configuration, gracefully shut down old worker processes.

- reopen — reopen log files

- -t — test the configuration file: nginx checks the configuration for correct syntax, and then tries to open files referred in the configuration.

- -T — same as -t , but additionally dump configuration files to standard output (1.9.2).

- -v — print nginx version.

- -V — print nginx version, compiler version, and configure parameters.

---

## Controlling nginx

**Source**: https://nginx.org/en/docs/control.html  

### Controlling nginx
Changing ConfigurationRotating Log-filesUpgrading Executable on the Fly
nginx can be controlled with signals. The process ID of the master process is written to the file /usr/local/nginx/logs/nginx.pid by default. This name may be changed at configuration time, or in nginx.conf using the pid directive. The master process supports the following signals:
 

TERM, INTfast shutdown
QUITgraceful shutdown
HUPchanging configuration,
keeping up with a changed time zone (only for FreeBSD and Linux),
starting new worker processes with a new configuration,
graceful shutdown of old worker processes
USR1re-opening log files
USR2upgrading an executable file
WINCHgraceful shutdown of worker processes

Individual worker processes can be controlled with signals as well, though it is not required. The supported signals are:
 

TERM, INTfast shutdown
QUITgraceful shutdown
USR1re-opening log files
WINCHabnormal termination for debugging
(requires debug_points to be enabled)

#### Changing Configuration

In order for nginx to re-read the configuration file, a HUP signal should be sent to the master process. The master process first checks the syntax validity, then tries to apply new configuration, that is, to open log files and new listen sockets. If this fails, it rolls back changes and continues to work with old configuration. If this succeeds, it starts new worker processes, and sends messages to old worker processes requesting them to shut down gracefully. Old worker processes close listen sockets and continue to service old clients. After all clients are serviced, old worker processes are shut down.

Let’s illustrate this by example. Imagine that nginx is run on FreeBSD and the command
 
```nginx

ps axw -o pid,ppid,user,%cpu,vsz,wchan,command | egrep '(nginx|PID)'

```

produces the following output:
 
```nginx

  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
33127 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33128 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)
33129 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)

```

If HUP is sent to the master process, the output becomes:
 
```nginx

  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33129 33126 nobody   0.0  1380 kqread nginx: worker process is shutting down (nginx)
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)

```

One of the old worker processes with PID 33129 still continues to work. After some time it exits:
 
```nginx

  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (n

---

## QUIC and HTTP/3 support

**Source**: https://nginx.org/en/docs/quic.html  

### Support for QUIC and HTTP/3
Building from sourcesConfiguration tipsTroubleshooting
Support for QUIC and HTTP/3 protocols is available since 1.25.0, it is included in Linux binary packages . Please refer to the ngx_http_v3_module documentation.

#### Building from sources

The build is configured using the configure command. Please refer to Building nginx from Sources for details.

The OpenSSL library version 3.5.1 or higher is recommended to build nginx with QUIC support. Otherwise, the OpenSSL compatibility layer will be used that does not support early data . Alternatively, BoringSSL , LibreSSL , or QuicTLS prebuilt libraries can be used.

Use the following command to configure nginx with BoringSSL :
 
```nginx

./configure
    --with-debug
    --with-http_v3_module
    --with-cc-opt="-I../boringssl/include"
    --with-ld-opt="-L../boringssl/build -lstdc++"

```

Alternatively, nginx can be configured with QuicTLS :
 
```nginx

./configure
    --with-debug
    --with-http_v3_module
    --with-cc-opt="-I../quictls/build/include"
    --with-ld-opt="-L../quictls/build/lib"

```

Alternatively, nginx can be configured with LibreSSL :
 
```nginx

./configure
    --with-debug
    --with-http_v3_module
    --with-cc-opt="-I../libressl/build/include"
    --with-ld-opt="-L../libressl/build/lib"

```

After configuration, nginx is compiled and installed using make .

#### Configuration tips

The listen directive in ngx_http_core_module module got a new parameter quic which enables HTTP/3 over QUIC on the specified port.

Along with the quic parameter it is also possible to specify the reuseport parameter to make it work properly with multiple workers.

To enable address validation:
 
```nginx

quic_retry on;

```

To enable 0-RTT:
 
```nginx

ssl_early_data on;

```

To enable GSO (Generic Segmentation Offloading):
 
```nginx

quic_gso on;

```

To set host key for various tokens:
 
```nginx

quic_host_key <filename>;

```

QUIC requires TLSv1.3 protocol version which is enabled by default in the ssl_protocols directive.

By default, GSO Linux-specific optimization is disabled. Enable it in case a corresponding network interface is configured to support GSO.

#### Troubleshooting

Tips that may help to identify problems:
 

- Ensure nginx is built with the proper SSL library.

- Ensure nginx is using the proper SSL library in runtime (the nginx -V shows what it is currently used).

- Ensure a client is actually sending requests over QUIC. It is recommended to start with a simple console client such as ngtcp2 to ensure the server is configured properly before trying with real browsers that may be quite picky with certificates.

- Build nginx with debug support and check the debug log. It should contain all details about the connection and why it failed. All related messages contain the “ quic ” prefix and can be easily filtered out.

- For a deeper investigation, additional debugging can be enabled using the following macros: NGX_QUIC_DEBUG_PACKETS , 

---

## Server names

**Source**: https://nginx.org/en/docs/http/server_names.html  

### Server names
Wildcard namesRegular expressions namesMiscellaneous namesInternationalized namesVirtual server selectionOptimizationCompatibility
Server names are defined using the server_name directive and determine which server block is used for a given request. See also “ How nginx processes a request ”. They may be defined using exact names, wildcard names, or regular expressions:
 
```nginx

server {
    listen       80;
    server_name  example.org  www.example.org;
    ...
}

server {
    listen       80;
    server_name  *.example.org;
    ...
}

server {
    listen       80;
    server_name  mail.*;
    ...
}

server {
    listen       80;
    server_name  ~^(?<user>.+)\.example\.net$;
    ...
}

```

The search is performed in the following order of priority and terminates on the first matching variant:
 

- the exact name

- the longest wildcard name starting with an asterisk, e.g. “ *.example.org ”

- the longest wildcard name ending with an asterisk, e.g. “ mail.* ”

- the first matching regular expression (in order of appearance in the configuration file)

#### Wildcard names

A wildcard name may contain an asterisk only on the name’s start or end, and only on a dot border. The names “ www.*.example.org ” and “ w*.example.org ” are invalid. However, these names can be specified using regular expressions, for example, “ ~^www\..+\.example\.org$ ” and “ ~^w.*\.example\.org$ ”. An asterisk can match several name parts. The name “ *.example.org ” matches not only www.example.org but www.sub.example.org as well.

A special wildcard name in the form “ .example.org ” can be used to match both the exact name “ example.org ” and the wildcard name “ *.example.org ”.

#### Regular expressions names

The regular expressions used by nginx are compatible with those used by the Perl programming language (PCRE). To use a regular expression, the server name must start with the tilde character:
 
```nginx

server_name  ~^www\d+\.example\.net$;

```

otherwise it will be treated as an exact name, or if the expression contains an asterisk, as a wildcard name (and most likely as an invalid one). Do not forget to set “ ^ ” and “ $ ” anchors. They are not required syntactically, but logically. Also note that domain name dots should be escaped with a backslash. A regular expression containing the characters “ { ” and “ } ” should be quoted:
 
```nginx

server_name  "~^(?<name>\w\d{1,3}+)\.example\.net$";

```

otherwise nginx will fail to start and display the error message:
 
```nginx

directive "server_name" is not terminated by ";" in ...

```

A named regular expression capture can be used later as a variable:
 
```nginx

server {
    server_name   ~^(www\.)?(?<domain>.+)$;

    location / {
        root   /sites/$domain;
    }
}

```

The PCRE library supports named captures using the following syntax: ? Perl 5.10 compatible syntax, supported since PCRE-7.0 ?' name ' Perl 5.10 compatible syntax, supported since PCRE-7.0 ?P Python compatible syntax, 

---

## How nginx processes a request

**Source**: https://nginx.org/en/docs/http/request_processing.html  

### How nginx processes a request
How to prevent processing requests with undefined server namesMixed name-based and IP-based virtual serversA simple PHP site configuration
#### Name-based virtual servers

nginx first decides which server should process the request. Let’s start with a simple configuration where all three virtual servers listen on port *:80:
 
```nginx

server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}

```

In this configuration nginx tests only the request’s header field “Host” to determine which server the request should be routed to. If its value does not match any server name, or the request does not contain this header field at all, then nginx will route the request to the default server for this port. In the configuration above, the default server is the first one — which is nginx’s standard default behaviour. It can also be set explicitly which server should be default, with the default_server parameter in the listen directive:
 
```nginx

server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}

```

 
The `default_server` parameter has been available since
version 0.8.21.
In earlier versions the `default` parameter should be used
instead.

Note that the default server is a property of the listen port and not of the server name. More about this later.

#### How to prevent processing requests with undefined server names

If requests without the “Host” header field should not be allowed, a server that just drops the requests can be defined:
 
```nginx

server {
    listen      80;
    server_name "";
    return      444;
}

```

Here, the server name is set to an empty string that will match requests without the “Host” header field, and a special nginx’s non-standard code 444 is returned that closes the connection.
 
Since version 0.8.48, this is the default setting for the
server name, so the `server_name ""` can be omitted.
In earlier versions, the machine’s hostname was used as
a default server name.

#### Mixed name-based and IP-based virtual servers

Let’s look at a more complex configuration where some virtual servers listen on different addresses:
 
```nginx

server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    ...
}

```

In this configuration, nginx first tests the IP address and port of the request against the listen directives of the server blocks. It then tests the “Host” header field of the request against the server_name entries of the server blocks that matched the IP address and port. If the server name is not found, the request will be pro

---

## Using nginx as HTTP load balancer

**Source**: https://nginx.org/en/docs/http/load_balancing.html  

### Using nginx as HTTP load balancer
Load balancing methodsDefault load balancing configurationLeast connected load balancingLeast time load balancingSession persistenceWeighted load balancingHealth checksFurther reading
#### Introduction

Load balancing across multiple application instances is a commonly used technique for optimizing resource utilization, maximizing throughput, reducing latency, and ensuring fault-tolerant configurations.

It is possible to use nginx as a very efficient HTTP load balancer to distribute traffic to several application servers and to improve performance, scalability and reliability of web applications with nginx.

#### Load balancing methods

The following load balancing mechanisms (or methods) are supported in nginx:
 

- round-robin — requests to the application servers are distributed in a round-robin fashion,

- least-connected — next request is assigned to the server with the least number of active connections,

- ip-hash — a hash-function is used to determine what server should be selected for the next request (based on the client’s IP address).

#### Default load balancing configuration

The simplest configuration for load balancing with nginx may look like the following:
 
```nginx

http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}

```

In the example above, there are 3 instances of the same application running on srv1-srv3. When the load balancing method is not specifically configured, it defaults to round-robin. All requests are proxied to the server group myapp1, and nginx applies HTTP load balancing to distribute the requests.

Reverse proxy implementation in nginx includes load balancing for HTTP, HTTPS, FastCGI, uwsgi, SCGI, memcached, and gRPC.

To configure load balancing for HTTPS instead of HTTP, just use “https” as the protocol.

When setting up load balancing for FastCGI, uwsgi, SCGI, memcached, or gRPC, use fastcgi_pass , uwsgi_pass , scgi_pass , memcached_pass , and grpc_pass directives respectively.

#### Least connected load balancing

Another load balancing discipline is least-connected. Least-connected allows controlling the load on application instances more fairly in a situation when some of the requests take longer to complete.

With the least-connected load balancing, nginx will try not to overload a busy application server with excessive requests, distributing the new requests to a less busy server instead.

Least-connected load balancing in nginx is activated when the least_conn directive is used as part of the server group configuration:
 
```nginx

    upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

```

#### Least time load balancing

Another load balancing discipline is least-time. Least-time

---

## Configuring HTTPS servers

**Source**: https://nginx.org/en/docs/http/configuring_https_servers.html  

### Configuring HTTPS servers
HTTPS server optimizationSSL certificate chainsA single HTTP/HTTPS serverName-based HTTPS servers     An SSL certificate with several names     Server Name IndicationCompatibility
To configure an HTTPS server, the ssl parameter must be enabled on listening sockets in the server block, and the locations of the server certificate and private key files should be specified:
 
```nginx

server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}

```

The server certificate is a public entity. It is sent to every client that connects to the server. The private key is a secure entity and should be stored in a file with restricted access, however, it must be readable by nginx’s master process. The private key may alternately be stored in the same file as the certificate:
 
```nginx

    ssl_certificate     www.example.com.cert;
    ssl_certificate_key www.example.com.cert;

```

in which case the file access rights should also be restricted. Although the certificate and the key are stored in one file, only the certificate is sent to a client.

The directives ssl_protocols and ssl_ciphers can be used to limit connections to include only the strong versions and ciphers of SSL/TLS. By default nginx uses “ ssl_protocols TLSv1.2 TLSv1.3 ” and “ ssl_ciphers HIGH:!aNULL:!MD5 ”, so configuring them explicitly is generally not needed. Note that default values of these directives were changed several times.

#### HTTPS server optimization

SSL operations consume extra CPU resources. On multi-processor systems several worker processes should be run, no less than the number of available CPU cores. The most CPU-intensive operation is the SSL handshake. There are two ways to minimize the number of these operations per client: the first is by enabling keepalive connections to send several requests via one connection and the second is to reuse SSL session parameters to avoid SSL handshakes for parallel and subsequent connections. The sessions are stored in an SSL session cache shared between workers and configured by the ssl_session_cache directive. One megabyte of the cache contains about 4000 sessions. The default cache timeout is 5 minutes. It can be increased by using the ssl_session_timeout directive. Here is a sample configuration optimized for a multi-core system with 10 megabyte shared session cache:
 
```nginx

worker_processes auto;

http {
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    server {
        listen              443 ssl;
        server_name         www.example.com;
        keepalive_timeout   70;

        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;
       

---

## How nginx processes a TCP/UDP session

**Source**: https://nginx.org/en/docs/stream/stream_processing.html  

### How nginx processes a TCP/UDP session

A TCP/UDP session from a client is processed in successive steps called phases :
 

`Post-accept`

The first phase after accepting a client connection.
The ngx_stream_realip_module
module is invoked at this phase.

`Pre-access`

Preliminary check for access.
The
ngx_stream_limit_conn_module
and
ngx_stream_set_module
modules are invoked at this phase.

`Access`

Client access limitation before actual data processing.
At this phase,
the ngx_stream_access_module
module is invoked,
for njs,
the js_access directive
is invoked.

`SSL`

TLS/SSL termination.
The ngx_stream_ssl_module
module is invoked at this phase.

`Preread`

Reading initial bytes of data into the
preread buffer
to allow modules such as
ngx_stream_ssl_preread_module
analyze the data before its processing.
For njs,
the js_preread directive
is invoked at this phase.

`Content`

Mandatory phase where data is actually processed, usually
proxied to
upstream servers,
or a specified value
is returned to a client.
For njs,
the js_filter directive
is invoked at this phase.

`Log`

The final phase
where the result of a client session processing is recorded.
The ngx_stream_log_module
module is invoked at this phase.

---
