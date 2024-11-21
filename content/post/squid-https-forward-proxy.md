---
title: "Configure Squid as an HTTPS Forward Proxy with Authentication"
date: 2024-11-21T19:11:23+08:00
draft: false
summary: A guide to configure Squid as an HTTPS forward proxy with authentication to resolve malformed HTTP response issue while downloading Go modules.
tags: ["Squid", "HTTPS", "Proxy", "Authentication", "Go"]
categories: ["English"]
---

Recently, I met an issue complaining malformed HTTP response while downloading Go modules behind an HTTPS proxy on macOS.

The issue occur after I set the environment variable `https_proxy` to an **HTTPS proxy** which has an **authentication**.

```shell
$ export https_proxy=https://user:password@proxy_server.com:443

$ go mod download -x golang.org/x/sync@latest

# get https://proxy.golang.org/golang.org/x/sync/@v/list
# get https://proxy.golang.org/golang.org/x/sync/@v/list: Get "https://proxy.golang.org/golang.org/x/sync/@v/list": malformed HTTP response "\x00\x00\f\x04\x00\x00\x00\x00\x00\x00\x03\x00\x00\x00d\x00\x04\x00\x00\xff\xff"
go: module golang.org/x/sync: Get "https://proxy.golang.org/golang.org/x/sync/@v/list": malformed HTTP response "\x00\x00\f\x04\x00\x00\x00\x00\x00\x00\x03\x00\x00\x00d\x00\x04\x00\x00\xff\xff"

```

Finally, I found a workaround.

1. Use Squid as an intermediate proxy server.
2. Set the environment variable `https_proxy` to the address of Squid **WITHOUT** authentication.
3. Configure Squid to forward requests to the original HTTPS proxy which has an authentication.

## Squid Configuration

### Environment

* **macOS** Sequoia 15.1
* **Go** 1.23.3 

### Steps

1. Edit the `$(brew --prefix)/etc/squid.conf` using the following content.  
    (**NOTE:** Remember to replace `<https_proxy_server>`, `<port>`, `<user>` and `<password>` with actual values.)

```squid.conf
http_port 3128

acl local src localhost
http_access allow local
http_access deny all

cache_peer <https_proxy_server> parent <port> 0 no-query no-digest login=<user>:<password> ssl
never_direct allow all
```

2. Restart Squid.

```shell
# Install the launchd service
# brew tap homebrew/services

brew services restart squid
```

3. Set the environment variable `https_proxy` to the address of Squid.

```shell
export https_proxy=http://127.0.0.1:3128
```

4. Test again.

```shell
$ go mod download -x golang.org/x/sync@latest

# get https://proxy.golang.org/golang.org/x/sync/@v/list
# get https://proxy.golang.org/golang.org/x/sync/@v/list: 200 OK (1.187s)
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.info
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.info: 200 OK (0.157s)
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.mod
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.mod: 200 OK (0.168s)
# get https://proxy.golang.org/sumdb/sum.golang.org/supported
# get https://proxy.golang.org/sumdb/sum.golang.org/supported: 404 Not Found (0.161s)
# get https://sum.golang.org/lookup/golang.org/x/sync@v0.9.0
# get https://sum.golang.org/lookup/golang.org/x/sync@v0.9.0: 200 OK (1.026s)
# get https://sum.golang.org/tile/8/0/x123/009
# get https://sum.golang.org/tile/8/1/488.p/229
# get https://sum.golang.org/tile/8/1/480
# get https://sum.golang.org/tile/8/3/000.p/1
# get https://sum.golang.org/tile/8/2/001.p/232
# get https://sum.golang.org/tile/8/0/x125/157.p/6
# get https://sum.golang.org/tile/8/1/488.p/229: 200 OK (0.234s)
# get https://sum.golang.org/tile/8/0/x125/157.p/6: 200 OK (0.234s)
# get https://sum.golang.org/tile/8/2/001.p/232: 200 OK (0.235s)
# get https://sum.golang.org/tile/8/1/480: 200 OK (0.306s)
# get https://sum.golang.org/tile/8/3/000.p/1: 200 OK (0.310s)
# get https://sum.golang.org/tile/8/0/x123/009: 200 OK (0.310s)
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.zip
# get https://proxy.golang.org/golang.org/x/sync/@v/v0.9.0.zip: 200 OK (0.155s)
```

It works!

