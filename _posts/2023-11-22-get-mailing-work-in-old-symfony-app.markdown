---
layout: post
title:  "Get Mails to work in an old Symfony project running on Docker"
date:   2023-11-22 12:00:00 +0100
categories: symfony mail hetzner docker
---

## Situation
We have a Symfony 2.6 application, which runs in a docker Container. This container was taken from one server and needed to be ported to a new hoster.

After porting the container to the new server, Smoke tests revealed that no E-Mails were sent. This is where the search starts.

## Test outgoing smtp conenction
### Hetzner VM:

Could this be the issue?:
https://blog.hqcodeshop.fi/archives/553-Hetzner-outgoing-mail-SMTP-blocked-on-TCP25.html

Trying to connect with sandbox.smtp.mailtrap.io on port 25:

```bash
bmt@docker-ce-ubuntu-4gb-nbg1-1:~$ telnet sandbox.smtp.mailtrap.io 25
Trying 3.209.246.195...
```

but with no response. Seems that those ports are actually closed. When opening the support site as stated in the blogpost, we have this option to to select regarding mails, but it seems we are not allowed to have them open.

![Alt text](../assets/img/hetzner_support.png)

So next option would be using SSL TLS on port 587. This seems to work:

```bash
bmt@docker-ce-ubuntu-4gb-nbg1-1:~$ openssl s_client -connect sandbox.smtp.mailtrap.io:587 -starttls smtp -crlf
CONNECTED(00000003)
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
verify return:1
depth=0 CN = *.smtp.mailtrap.io
verify return:1
---
Certificate chain
 0 s:CN = *.smtp.mailtrap.io
   i:C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
 1 s:C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
   i:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
 2 s:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
   i:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
 3 s:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
   i:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIGQDCCBSigAwIBAgIRAMYY1yYe6TzdJpEcTiR48UgwDQYJKoZIhvcNAQELBQAw
gY8xCzAJBgNVBAYTAkdCMRswGQYDVQQIExJHcmVhdGVyIE1hbmNoZXN0ZXIxEDAO
BgNVBAcTB1NhbGZvcmQxGDAWBgNVBAoTD1NlY3RpZ28gTGltaXRlZDE3MDUGA1UE
AxMuU2VjdGlnbyBSU0EgRG9tYWluIFZhbGlkYXRpb24gU2VjdXJlIFNlcnZlciBD
QTAeFw0yMjEyMjAwMDAwMDBaFw0yNDAxMjAyMzU5NTlaMB0xGzAZBgNVBAMMEiou
c210cC5tYWlsdHJhcC5pbzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
AMeKpSylmChHlOPTAAU3ZxIymiiq0/BR9r80aYlZ+egJ/ZgxquIX997lJuphj3Qn
tekCH/WjQkPpminquOX4AfEtZ49jWvPXY38fRXu+zgoqDi1q9AUIF5AEtyNIYdO4
zSCNB16ZkuXC6pdzT3xDMTZd5m46fXgpT8INyPyTuDxyNc3ZskPb/fWDImCx3+2Q
mKXbdsgO6UFm8qiXA5MbMQY3WIZ+5QIrEVUsaPoGc5ADxOgWh7+c8LtstFNVrVV8
mUIqGxRFb931RyqXDo9TDEi9Xo2aZURtqVbdBJoF5DPQ7OIfU/8SZjvslmOOazX8
BaTtFlDs+jCT21Ec8pUNJD8CAwEAAaOCAwYwggMCMB8GA1UdIwQYMBaAFI2MXsRU
rYrhd+mb+ZsF4bgBjWHhMB0GA1UdDgQWBBS+ruU6Ie+X7FYpEn3/rjkLcrsjKjAO
BgNVHQ8BAf8EBAMCBaAwDAYDVR0TAQH/BAIwADAdBgNVHSUEFjAUBggrBgEFBQcD
AQYIKwYBBQUHAwIwSQYDVR0gBEIwQDA0BgsrBgEEAbIxAQICBzAlMCMGCCsGAQUF
BwIBFhdodHRwczovL3NlY3RpZ28uY29tL0NQUzAIBgZngQwBAgEwgYQGCCsGAQUF
BwEBBHgwdjBPBggrBgEFBQcwAoZDaHR0cDovL2NydC5zZWN0aWdvLmNvbS9TZWN0
aWdvUlNBRG9tYWluVmFsaWRhdGlvblNlY3VyZVNlcnZlckNBLmNydDAjBggrBgEF
BQcwAYYXaHR0cDovL29jc3Auc2VjdGlnby5jb20wLwYDVR0RBCgwJoISKi5zbXRw
Lm1haWx0cmFwLmlvghBzbXRwLm1haWx0cmFwLmlvMIIBfgYKKwYBBAHWeQIEAgSC
AW4EggFqAWgAdgB2/4g/Crb7lVHCYcz1h7o0tKTNuyncaEIKn+ZnTFo6dAAAAYUw
M58HAAAEAwBHMEUCIQCd70IvUUqm7U3sHCGHFeEXGHN8cfGDoaZ9BpIuGp+fRgIg
afgSbWshixwEiyohE+TJ0So9d4EDOhPLpZ6NB4fZVb8AdgDatr9rP7W2Ip+bwrtc
a+hwkXFsu1GEhTS9pD0wSNf7qwAAAYUwM58XAAAEAwBHMEUCIQCriA1PHk6ol8RD
8hVrQt8wYAZYUi132lJVcwJss+8y9wIgWR50DSBLXU+xsw3x20HZjK0vTcLgbjX8
QG0r1yh15u0AdgDuzdBk1dsazsVct520zROiModGfLzs3sNRSFlGcR+1mwAAAYUw
M57oAAAEAwBHMEUCIQCGChNc4oUbZiJnD5EvM9bSP0Us55Psc8SUyhejxZ44zwIg
K/Uxg9L7O3P2fwMi6qnTVDEpPiV6qBUKxz+iYhDiji0wDQYJKoZIhvcNAQELBQAD
ggEBAHRiBa+wuRxVa0OcfM9Qgu/kFjfP4ZQtU7tjX7mMH4rxF7aMmpXfT25wq9fj
Z012y1B3aiHF9j2h5hrOYWp+BVg2ECovEpSjs2YcQChtLDKogkYK4RIqNKg9rofV
lRzZy5Npkif1RmrUInOUpds1WefBkx5GfvnFj8gOKpcepI+HLYECAVryCcYHadNM
lSnkzWJGwTFbg51e0Y195ckuQUQxgFdmMbeMNMqDVweiIkYn12gHyNVAAFE5CdKL
YDBrnWXEUMXP24QkoG6lCfX5NTlMoxBEYEIcutJcK1+7hUBy2XXdUts4yKaagFdQ
g7L8iEO5f0zk8sHx9gvgoFlcZKQ=
-----END CERTIFICATE-----
subject=CN = *.smtp.mailtrap.io

issuer=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 6429 bytes and written 429 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
250 STARTTLS
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: D2D4DFBF6AA3B2E22FFF48AD0DAA163840120AED739718F7D8A0956D7FA69874
    Session-ID-ctx: 
    Resumption PSK: 042221DE62FDD347B7B1CA2B55343FAAF4EAB1B735DB491BDE43DA5994DDDED972562072A7C63573A1F8DA5AECBE2C8F
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 26 60 be d8 36 bb 20 85-9a 43 cd 95 a5 67 d2 cd   &`..6. ..C...g..
    0010 - a2 fd d1 b1 b2 aa 03 56-6a b1 e3 72 22 a7 27 85   .......Vj..r".'.

    Start Time: 1700647885
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: 2B678BF6562D27A6528069B757E7913660DF9DA17FD8E573C7BACD7CD2425443
    Session-ID-ctx: 
    Resumption PSK: 8A1F2450DBD491CD98A1A546C0667C43F3A6452FAB884846078A36FDCFEB2E66E9AF22604AF4ED8BF18C21A4F9BFE408
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 53 9c fd 32 55 b4 52 09-07 64 05 09 33 ff aa ae   S..2U.R..d..3...
    0010 - b1 1d 70 00 3c fa d2 80-2d cc c9 3f 74 8d e3 3b   ..p.<...-..?t..;

    Start Time: 1700647885
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
```

This configuration seems to work. Lets give it a try on the docker container

## Testing SMTP connection from within Docker Container

```bash
root@hpbmt:/var/www/hp-bmt# openssl s_client -connect sandbox.smtp.mailtrap.io:587 -starttls smtp      
CONNECTED(00000003)
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA
verify return:1
depth=0 CN = *.smtp.mailtrap.io
verify return:1
---
Certificate chain
 0 s:/CN=*.smtp.mailtrap.io
   i:/C=GB/ST=Greater Manchester/L=Salford/O=Sectigo Limited/CN=Sectigo RSA Domain Validation Secure Server CA
 1 s:/C=GB/ST=Greater Manchester/L=Salford/O=Sectigo Limited/CN=Sectigo RSA Domain Validation Secure Server CA
   i:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
 2 s:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
   i:/C=GB/ST=Greater Manchester/L=Salford/O=Comodo CA Limited/CN=AAA Certificate Services
 3 s:/C=GB/ST=Greater Manchester/L=Salford/O=Comodo CA Limited/CN=AAA Certificate Services
   i:/C=GB/ST=Greater Manchester/L=Salford/O=Comodo CA Limited/CN=AAA Certificate Services
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIGQDCCBSigAwIBAgIRAMYY1yYe6TzdJpEcTiR48UgwDQYJKoZIhvcNAQELBQAw
gY8xCzAJBgNVBAYTAkdCMRswGQYDVQQIExJHcmVhdGVyIE1hbmNoZXN0ZXIxEDAO
BgNVBAcTB1NhbGZvcmQxGDAWBgNVBAoTD1NlY3RpZ28gTGltaXRlZDE3MDUGA1UE
AxMuU2VjdGlnbyBSU0EgRG9tYWluIFZhbGlkYXRpb24gU2VjdXJlIFNlcnZlciBD
QTAeFw0yMjEyMjAwMDAwMDBaFw0yNDAxMjAyMzU5NTlaMB0xGzAZBgNVBAMMEiou
c210cC5tYWlsdHJhcC5pbzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
AMeKpSylmChHlOPTAAU3ZxIymiiq0/BR9r80aYlZ+egJ/ZgxquIX997lJuphj3Qn
tekCH/WjQkPpminquOX4AfEtZ49jWvPXY38fRXu+zgoqDi1q9AUIF5AEtyNIYdO4
zSCNB16ZkuXC6pdzT3xDMTZd5m46fXgpT8INyPyTuDxyNc3ZskPb/fWDImCx3+2Q
mKXbdsgO6UFm8qiXA5MbMQY3WIZ+5QIrEVUsaPoGc5ADxOgWh7+c8LtstFNVrVV8
mUIqGxRFb931RyqXDo9TDEi9Xo2aZURtqVbdBJoF5DPQ7OIfU/8SZjvslmOOazX8
BaTtFlDs+jCT21Ec8pUNJD8CAwEAAaOCAwYwggMCMB8GA1UdIwQYMBaAFI2MXsRU
rYrhd+mb+ZsF4bgBjWHhMB0GA1UdDgQWBBS+ruU6Ie+X7FYpEn3/rjkLcrsjKjAO
BgNVHQ8BAf8EBAMCBaAwDAYDVR0TAQH/BAIwADAdBgNVHSUEFjAUBggrBgEFBQcD
AQYIKwYBBQUHAwIwSQYDVR0gBEIwQDA0BgsrBgEEAbIxAQICBzAlMCMGCCsGAQUF
BwIBFhdodHRwczovL3NlY3RpZ28uY29tL0NQUzAIBgZngQwBAgEwgYQGCCsGAQUF
BwEBBHgwdjBPBggrBgEFBQcwAoZDaHR0cDovL2NydC5zZWN0aWdvLmNvbS9TZWN0
aWdvUlNBRG9tYWluVmFsaWRhdGlvblNlY3VyZVNlcnZlckNBLmNydDAjBggrBgEF
BQcwAYYXaHR0cDovL29jc3Auc2VjdGlnby5jb20wLwYDVR0RBCgwJoISKi5zbXRw
Lm1haWx0cmFwLmlvghBzbXRwLm1haWx0cmFwLmlvMIIBfgYKKwYBBAHWeQIEAgSC
AW4EggFqAWgAdgB2/4g/Crb7lVHCYcz1h7o0tKTNuyncaEIKn+ZnTFo6dAAAAYUw
M58HAAAEAwBHMEUCIQCd70IvUUqm7U3sHCGHFeEXGHN8cfGDoaZ9BpIuGp+fRgIg
afgSbWshixwEiyohE+TJ0So9d4EDOhPLpZ6NB4fZVb8AdgDatr9rP7W2Ip+bwrtc
a+hwkXFsu1GEhTS9pD0wSNf7qwAAAYUwM58XAAAEAwBHMEUCIQCriA1PHk6ol8RD
8hVrQt8wYAZYUi132lJVcwJss+8y9wIgWR50DSBLXU+xsw3x20HZjK0vTcLgbjX8
QG0r1yh15u0AdgDuzdBk1dsazsVct520zROiModGfLzs3sNRSFlGcR+1mwAAAYUw
M57oAAAEAwBHMEUCIQCGChNc4oUbZiJnD5EvM9bSP0Us55Psc8SUyhejxZ44zwIg
K/Uxg9L7O3P2fwMi6qnTVDEpPiV6qBUKxz+iYhDiji0wDQYJKoZIhvcNAQELBQAD
ggEBAHRiBa+wuRxVa0OcfM9Qgu/kFjfP4ZQtU7tjX7mMH4rxF7aMmpXfT25wq9fj
Z012y1B3aiHF9j2h5hrOYWp+BVg2ECovEpSjs2YcQChtLDKogkYK4RIqNKg9rofV
lRzZy5Npkif1RmrUInOUpds1WefBkx5GfvnFj8gOKpcepI+HLYECAVryCcYHadNM
lSnkzWJGwTFbg51e0Y195ckuQUQxgFdmMbeMNMqDVweiIkYn12gHyNVAAFE5CdKL
YDBrnWXEUMXP24QkoG6lCfX5NTlMoxBEYEIcutJcK1+7hUBy2XXdUts4yKaagFdQ
g7L8iEO5f0zk8sHx9gvgoFlcZKQ=
-----END CERTIFICATE-----
subject=/CN=*.smtp.mailtrap.io
issuer=/C=GB/ST=Greater Manchester/L=Salford/O=Sectigo Limited/CN=Sectigo RSA Domain Validation Secure Server CA
---
No client certificate CA names sent
Peer signing digest: SHA256
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 6345 bytes and written 302 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 001735E42F41FEB58BC10A2099A90BD6B4C7AE9A95DF89FF4FE86EE93CE39C03
    Session-ID-ctx: 
    Master-Key: 059189EB369C6A3974F4BA40D3DE786833AD59203C1FBDC0D0092B27AC3B4234BB1435DA2EE9EABDC84D58B5C1480A79
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1700657858
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: yes
---
250 STARTTLS
```

It seems to connect as well, although only with TLS 1.2, not with 1.3. Lets try to use this in the Symfony configuration:

app/config/config.yml
```yaml
# Swiftmailer Configuration
swiftmailer:
    port: 587
    transport: smtp
    host:      sandbox.smtp.mailtrap.io
    username:  a719b4f5c52719
    password:  my_super_duper_secret_password
    encryption: tls
    auth_mode: plain
    spool:     { type: memory }
```

Now lets try to send a mail:

```bash
root@hpbmt:/var/www/hp-bmt# php app/console swiftmailer:email:send --to "ping@tools.mxtoolbox.com" --from "ste.staub@gmail.com" --subject "Test"           
Body: Test
Sent 1 emails


                                                                                                     
  [Symfony\Component\Debug\Exception\ContextErrorException]                                          
  Warning: stream_socket_enable_crypto(): SSL operation failed with code 1. OpenSSL Error messages:  
  error:1409442E:SSL routines:ssl3_read_bytes:tlsv1 alert protocol version

```

It seems that the version of TLS is an issue here...

The container uses OpenSSL 1.1.0j, which is from 20 Nov 2018, and therefore only TLS up to v1.2 is available.

The docker container is running Debian Stretch with php version 5.6 which should already support TLS1.2. So the question remains, why does it try to connect with tlsv1 and how could we force tlsv1.2?

### Investigating Swift Mailer

Turns out, swiftmailer hardcoded use of STREAM_CRYPTO_METHOD_TLS_CLIENT when creating a tls socket. This defaults to TLSv1.0 which is not supported by todays mail clients anymore.

This was fixed for swiftmailer 6.0.3 with this commit: 

https://github.com/Rotzbua/swiftmailer/commit/f79c328c606ad9b2f369092cfd69c4b512df75cd

As we are using version 5.3.1 for swiftmailer, we are stuck with TLSv1.0.

## Options

- Upgrade Swiftmailer
- Send mails through an unencrypted channel using a port other than 25
- Use a seperate docker container as a internal mail server which then forwards mails to an actual mailserver through a secure channel