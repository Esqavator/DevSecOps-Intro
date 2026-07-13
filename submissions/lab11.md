# Lab 11 — Reverse Proxy Hardening

## Overview

The OWASP Juice Shop application is placed behind a hardened Nginx reverse proxy.

The main architecture is:

~~~text
Client
  |
  +-- http://localhost:8080
  |       |
  |       +-- 308 redirect to HTTPS
  |
  +-- https://localhost:8443
          |
          +-- hardened Nginx
                  |
                  +-- Juice Shop:3000
~~~

The bonus WAF architecture is:

~~~text
Client
  |
  +-- http://localhost:8082
          |
          +-- ModSecurity v3 + OWASP CRS v4
                  |
                  +-- https://nginx:8443
                          |
                          +-- Juice Shop:3000
~~~

## Task 1 — HTTPS and security headers

### HTTP to HTTPS redirect

Nginx listens on port `8080` and permanently redirects all requests to HTTPS on port `8443`.

Observed result:

~~~text
HTTP/1.1 308 Permanent Redirect
Location: https://localhost:8443/
~~~

### TLS configuration

The reverse proxy accepts only TLS 1.3:

~~~nginx
ssl_protocols TLSv1.3;
~~~

Observed negotiation:

~~~text
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer Temp Key: X25519, 253 bits
~~~

A TLS 1.2 connection was rejected with a protocol-version alert.

The certificate is self-signed because this is a local laboratory environment. It contains SAN entries for `localhost`, `juice.local`, and `127.0.0.1`.

### Security headers

The following headers are added by Nginx:

1. `Strict-Transport-Security`
2. `X-Content-Type-Options`
3. `X-Frame-Options`
4. `Referrer-Policy`
5. `Permissions-Policy`
6. `Content-Security-Policy-Report-Only`

Observed values:

~~~text
strict-transport-security: max-age=63072000; includeSubDomains; preload
x-content-type-options: nosniff
x-frame-options: DENY
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), microphone=(), geolocation=()
content-security-policy-report-only: default-src 'self'; ...
~~~

CSP is initially deployed in report-only mode. This allows violations and incompatible application resources to be discovered before enforcement is enabled.

## Task 2 — Reverse proxy hardening

### Rate limiting

Requests to the authentication endpoint are limited by client IP:

~~~nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
limit_req zone=login burst=5 nodelay;
limit_req_status 429;
~~~

During a parallel test with 60 login attempts, the result was:

~~~text
6 401
54 429
~~~

This confirms that requests above the configured limit are rejected with HTTP `429 Too Many Requests`.

### Connection limiting

The number of simultaneous connections per client IP is limited:

~~~nginx
limit_conn_zone $binary_remote_addr zone=conn:10m;
limit_conn conn 50;
~~~

### Timeout protection

The following timeout values are configured:

~~~nginx
client_body_timeout 10s;
client_header_timeout 10s;
send_timeout 10s;
keepalive_timeout 10s;

proxy_connect_timeout 5s;
proxy_read_timeout 30s;
proxy_send_timeout 30s;
~~~

A TLS client sent incomplete HTTP headers and kept the connection open.

Observed result:

~~~text
Connection closed by Nginx without a response.
Elapsed seconds: 9.88
~~~

This demonstrates protection against slow-header and Slowloris-style connections.

### Cipher suites and curves

TLS 1.3 cipher suites are explicitly restricted:

~~~nginx
ssl_conf_command Ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;
ssl_ecdh_curve X25519:secp384r1;
~~~

Observed connection:

~~~text
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer Temp Key: X25519, 253 bits
~~~

### TLS session hardening

~~~nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
~~~

Session tickets are disabled so that ticket-key rotation is not required and old ticket keys cannot be used to recover previous TLS sessions.

## Certificate rotation

For production, certificates should be issued by an automated ACME client or an internal PKI.

A safe rotation procedure is:

1. Request or generate a new certificate and private key.
2. Verify the certificate hostname, SAN values, expiration date, and certificate chain.
3. Store the new files with restrictive permissions.
4. Replace the certificate files atomically.
5. Run `nginx -t`.
6. Reload Nginx with `nginx -s reload`.
7. Verify the new certificate using `openssl s_client`.
8. Retain the previous certificate briefly to allow rollback.

Reloading Nginx is preferable to restarting it because existing connections can finish while new workers load the updated certificate.

Private keys must not be committed to Git. The local certificate directory is excluded using `.gitignore`.

## OCSP stapling

OCSP stapling is not enabled for the laboratory certificate because it is self-signed and has no public certificate authority or OCSP responder.

In production, OCSP stapling can be enabled with:

~~~nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /path/to/full-chain.pem;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;
~~~

The server periodically retrieves a signed certificate-status response from the CA and sends it during the TLS handshake. This improves privacy and reduces the need for each client to contact the CA directly.

The full issuer chain and a reachable OCSP responder are required.

## Bonus — ModSecurity and OWASP CRS

The WAF uses:

~~~text
Image: owasp/modsecurity-crs:4.25.1-nginx-lts
ModSecurity: 3.0.16
ModSecurity Nginx connector: 1.0.4
OWASP CRS: 4.25.1
Blocking Paranoia Level: 1
Detection Paranoia Level: 1
Inbound anomaly threshold: 5
~~~

The WAF runs in blocking mode:

~~~yaml
MODSEC_RULE_ENGINE: "On"
BLOCKING_PARANOIA: "1"
DETECTION_PARANOIA: "1"
ANOMALY_INBOUND: "5"
~~~

### WAF verification

The same SQL injection payload was sent directly to Nginx and through the WAF:

~~~text
1' OR '1'='1
~~~

Direct request:

~~~text
https://localhost:8443/rest/products/search
HTTP status: 200
~~~

Request through ModSecurity:

~~~text
http://localhost:8082/rest/products/search
HTTP status: 403
~~~

The audit log recorded:

~~~text
Rule 942100 — SQL Injection Attack Detected via libinjection
Rule 949110 — Inbound Anomaly Score Exceeded
Total inbound anomaly score: 5
HTTP response: 403
~~~

Therefore, the request is accepted by the ordinary reverse proxy but blocked when it passes through ModSecurity and OWASP CRS.

## WAF trade-offs

A WAF provides an additional security layer, but it does not replace secure application code.

Main trade-offs include:

- false positives that may block legitimate requests;
- additional latency and CPU usage;
- larger log volume;
- the need to update CRS regularly;
- application-specific rule exclusions;
- testing requirements before increasing the paranoia level.

Paranoia Level 1 was selected because it provides baseline protection with a relatively low false-positive rate.

In production, WAF rules should first be evaluated in detection mode. After reviewing audit logs, narrowly scoped exclusions can be created for known legitimate requests. Entire rule groups should not be disabled unless strictly necessary.

## Verification summary

| Control | Result |
|---|---|
| HTTP to HTTPS redirect | Passed — HTTP 308 |
| TLS 1.3 only | Passed |
| TLS 1.2 rejected | Passed |
| Security headers | Passed — 6 headers |
| Login rate limiting | Passed — HTTP 429 |
| Connection limiting | Configured |
| Slow-header timeout | Passed — connection closed after 9.88 seconds |
| Restricted TLS ciphers | Passed |
| X25519 key exchange | Passed |
| Session tickets disabled | Passed |
| ModSecurity enabled | Passed |
| OWASP CRS v4 enabled | Passed |
| WAF blocking test | Passed — HTTP 403 |
| Audit rule ID | 942100 |
| Blocking rule ID | 949110 |
