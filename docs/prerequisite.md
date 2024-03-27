Please install the following softwares.

1. [Java 17](https://bell-sw.com/pages/downloads/#jdk-17-lts) or later
1. [CF CLI](https://github.com/cloudfoundry/cli)
    ```bash
    brew install cloudfoundry/tap/cf-cli
    ```
    ```
    $ cf -v
    cf version 8.7.5+8aa8625.2023-11-20
    ```
1. [curl](https://curl.se/)
    ```bash
    brew install curl
    ```
    ```
    $ curl --version
    curl 8.4.0 (x86_64-apple-darwin22.0) libcurl/8.4.0 (SecureTransport) LibreSSL/3.3.6 zlib/1.2.11 nghttp2/1.51.0
    Release-Date: 2023-10-11
    Protocols: dict file ftp ftps gopher gophers http https imap imaps ldap ldaps mqtt pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
    Features: alt-svc AsynchDNS GSS-API HSTS HTTP2 HTTPS-proxy IPv6 Kerberos Largefile libz MultiSSL NTLM NTLM_WB SPNEGO SSL threadsafe UnixSockets
    ```
1. [jq](https://jqlang.github.io/jq/)
    ```bash
    brew install jq
    ```
    ```
    $ jq --version
    jq-1.7.1
    ```