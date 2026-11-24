---
description: "How F5 NGINX Plus meets FIPS 140-2 and FIPS 140-3 Level 1 with a FIPS-validated OpenSSL library, and how to check that it runs in FIPS mode."
title: NGINX Plus FIPS Compliance
toc: true
weight: 600
f5-content-type: concept
f5-product: F5 NGINX Plus
f5-keywords: "FIPS 140-2, FIPS 140-3, FIPS mode, FIPS compliance, OpenSSL FIPS provider, CMVP, cipher suites, RHEL"
f5-summary: >
  F5 NGINX Plus is FIPS 140-2 Level 1 and FIPS 140-3 Level 1 compliant when the operating system and its OpenSSL library run in FIPS mode.
  NGINX Plus uses the FIPS-validated OpenSSL library of the operating system for all its cryptography.
  This page explains FIPS, shows how to check that the operating system, OpenSSL, and NGINX Plus run in FIPS mode, and lists the ciphers that FIPS mode turns off.
f5-audience: operator
---

## What is FIPS

The Federal Information Processing Standard (FIPS) of the [U.S. National Institute of Standards and Technology](https://www.nist.gov/) (NIST) sets the security requirements for cryptographic modules in federal IT systems. [FIPS 140-2](https://csrc.nist.gov/pubs/fips/140-2/upd2/final) and its successor [FIPS 140-3](https://csrc.nist.gov/pubs/fips/140-3/final) set strict standards to protect sensitive information, such as government communications and citizen data.

## Why FIPS-140 matters

FIPS 140 is a mandatory cryptographic standard in the United States and Canada, for federal agencies, their contractors, and many regulated industries.

Non-compliance can lead to lost contracts, restricted access to projects, or fines. In severe cases, it can lead to data breaches that expose personal information or put national security at risk.

Some industries, such as finance, healthcare, and energy, also adopt FIPS to improve data protection and operational security.

<span id="fips-compliance-in-us"></span>

### FIPS compliance in the United States

FIPS 140-2 is being phased out under the [FIPS 140-3 transition plan](https://csrc.nist.gov/projects/fips-140-3-transition-effort). Since September 22, 2026, all FIPS 140-2 certificates are on the Historical List of the Cryptographic Module Validation Program (CMVP). CMVP still supports buying and using these modules for existing systems. Federal agencies decide when they move to modules with FIPS 140-3 validation only.

{{< table >}}

| **Sector / Program**           | **Version**    | **Status**    |
|--------------------------------|----------------|---------------|
| **Federal Programs**           |                |               |
| CJIS                           | 140-2 or 140-3 | Mandatory     |
| FedRAMP                        | 140-2 or 140-3 | Mandatory     |
| FISMA                          | 140-2 or 140-3 | Mandatory     |
| DFARS                          | 140-2 or 140-3 | Mandatory     |
| DoDIN APL                      | 140-2 or 140-3 | Mandatory     |
| FAA                            | 140-2 to 140-3 | Transitioning |
| TSA                            | 140-2 or 140-3 | Recommended   |
| **Defense & Intelligence**     |                |               |
| CMMC                           | 140-2 or 140-3 | Mandatory     |
| Intelligence Community         | 140-2 to 140-3 | Transitioning |
| NSA CSfC                       | 140-2 to 140-3 | Transitioning |
| Military & Tactical Systems    | 140-2 to 140-3 | Transitioning |
| **Healthcare & Education**     |                |               |
| HIPAA                          | 140-2 or 140-3 | Mandatory     |
| HITECH                         | 140-2 or 140-3 | Mandatory     |
| Department of Veterans Affairs | 140-2 or 140-3 | Mandatory     |
| FERPA                          | 140-2 or 140-3 | Recommended   |
| **Commercial/Private Sector**  |                |               |
| PCI DSS                        | 140-2 or 140-3 | Recommended   |
| Common Criteria                | 140-2 or 140-3 | Recommended   |
| **Infrastructure & Critical Systems** |         |               |
| Critical Infrastructure        | 140-2 or 140-3 | Recommended   |
| Nuclear Regulatory Commission  | 140-2 or 140-3 | Recommended   |
| **State & Local Government**   |                |               |
| State and Local Gov Programs   | 140-2 or 140-3 | Mandatory     |

{{< /table >}}

### FIPS compliance in other countries

Although FIPS 140 is primarily a North American government cryptographic standard, it is widely recognized as a global benchmark for cryptographic security. Numerous countries outside North America align their cryptographic requirements with FIPS, especially in regulated sectors such as finance, defense, healthcare, and critical infrastructure.

{{< table >}}

| Country/Region | FIPS Use                                                                    |
|----------------|-----------------------------------------------------------------------------|
| Australia      | Referenced for government, defense, and cryptography systems.               |
| Canada         | Mandatory for federal and sensitive systems.                                |
| Denmark        | Referenced in finance, healthcare, and NATO communications.                 |
| Estonia        | Adopted for e-governance and critical systems.                              |
| Finland        | Relied on for defense and NATO communications.                              |
| France         | Relied on for defense and secure systems.                                   |
| Germany        | Relied on for defense, critical infrastructure, and NATO communications.    |
| Israel         | Trusted in defense, government, and financial systems.                      |
| Italy          | Relied on for defense and financial cryptography.                           |
| Japan          | Referenced in government and financial cryptographic practices.             |
| Netherlands    | Referenced in finance, healthcare, and NATO communications.                 |
| New Zealand    | Referenced for government and national cryptography.                        |
| Poland         | Relied on for secure government and NATO communications.                    |
| Spain          | Referenced in NATO communications and critical systems.                     |
| Sweden         | Relied on for defense and secure NATO communications.                       |
| UAE            | Trusted in finance, energy, and interoperability with the U.S. cryptography.|
| United Kingdom | Referenced for defense, health, and procurement standards.                  |
| United States  | Mandatory for federal government systems and contractors.                   |

{{< /table >}}

## FIPS compliant vs FIPS validated

FIPS validation is a formal process with several steps, which certifies cryptographic modules through testing under the [Cryptographic Module Validation Program](https://csrc.nist.gov/Projects/cryptographic-module-validation-program/cmvp-flow) (CMVP). [NIST](https://csrc.nist.gov/) manages the process, and accredited third-party laboratories test the cryptographic module. A module that passes is officially FIPS-validated, also called FIPS-certified.

FIPS compliance means that a system or a module claims to meet the FIPS requirements, but hasn't been officially tested or certified under the CMVP.

## FIPS compliance with NGINX Plus

NGINX Plus is **FIPS 140-2 Level 1** and **FIPS 140-3 Level 1 compliant**, when the operating system and the OpenSSL library run in FIPS mode.

NGINX Plus uses only the FIPS-validated OpenSSL library of the operating system for all its SSL/TLS, HTTP/2, and HTTP/3 encryption and decryption.

## FIPS compliance with NGINX Open Source

F5 tests NGINX Plus on operating systems in FIPS mode. NGINX Open Source isn't verified for such environments, especially with third-party builds, or with modules that implement their own cryptographic functions.

Compiling NGINX Open Source for FIPS mode can also need more dependencies on the operating system, which can bring unintended risks. Ask your security and compliance teams whether your configuration meets the FIPS requirements.

## FIPS validation of operating systems

Several operating system vendors have FIPS 140-2 Level 1 and FIPS 140-3 Level 1 validation for the OpenSSL cryptographic module of their operating systems:

- Red Hat: [Red Hat FIPS certifications](https://access.redhat.com/compliance/fips)
- Ubuntu: [Overview of FIPS-certified modules](https://documentation.ubuntu.com/security/docs/compliance/fips/fips-overview/)
- Oracle: [Oracle FIPS certifications](https://www.oracle.com/corporate/security-practices/assurance/development/external-security-evaluations/fips/certifications/)
- SUSE: [SUSE FIPS 140-3 cryptographic certificates](https://www.suse.com/c/suse-has-received-first-fips-140-3-cryptographic-certificates/)
- AWS: [FIPS 140-3 compliance](https://aws.amazon.com/compliance/fips/)
- Amazon Linux: [Achieving FIPS 140-3 validation](https://aws.amazon.com/blogs/compute/amazon-linux-2023-achieves-fips-140-3-validation/)

To check whether your operating system or cryptographic module is FIPS-validated, use the [NIST database search tool](https://csrc.nist.gov/Projects/cryptographic-module-validation-program/validated-modules/search).

## FIPS validation of OpenSSL

OpenSSL 3.0 and later have a FIPS provider, which runs the cryptographic operations in a FIPS-compliant mode.

- **FIPS 140-3**: The FIPS provider of OpenSSL 3.1.2 [has FIPS 140-3 validation](https://openssl-library.org/post/2025-03-11-fips-140-3/), under certificate [#4985](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4985).
- **FIPS 140-2**: The FIPS provider of OpenSSL 3.0.x has FIPS 140-2 validation, under certificates [#4811](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4811) and [#4282](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4282). NIST has moved both certificates to the Historical List.

<span id="verification-of-correct-operation-of-nginx-plus"></span>

## Verify that NGINX Plus runs in FIPS mode

These steps deploy NGINX Plus in a FIPS-compliant environment, and check that the FIPS operations work:

1. [Configure](#os-fips-setup) the operating system for FIPS mode, and [check](#os-fips-check) that it runs in FIPS mode.
1. [Check](#openssl-fips-check) that the OpenSSL library runs in FIPS mode.
1. [Install NGINX Plus](#nginx-plus-instll), and [check the ciphers](#nginx-plus-fips-check) that it accepts.

The steps use Red Hat Enterprise Linux (RHEL) 9.6 as an example. You can adapt them for other Linux operating systems that support FIPS mode.

### Step 1: Configure the operating system to use FIPS mode {#os-fips-setup}

The [Red Hat FIPS documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/switching-rhel-to-fips-mode_security-hardening) explains how to switch RHEL between FIPS mode and non-FIPS mode: you change the boot options and restart the system.

For other operating systems, see their documentation, for example:

- RHEL 9: [Switching to FIPS mode](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/switching-rhel-to-fips-mode_security-hardening)
- Ubuntu: [Switching to FIPS mode](https://documentation.ubuntu.com/security/docs/compliance/fips/how-to-install-ubuntu-with-fips/)
- SLES: [How to enable FIPS](https://www.suse.com/support/kb/doc/?id=000019432)
- Oracle Linux 9: [Configuring FIPS mode](https://docs.oracle.com/en/operating-systems/oracle-linux/9/security/configuring_fips_mode.html#configuring-fips-mode)
- Amazon Linux 2023: [Enabling FIPS mode](https://docs.aws.amazon.com/linux/al2023/ug/fips-mode.html)
- Amazon Linux 2: [Enabling FIPS mode](https://docs.aws.amazon.com/linux/al2/ug/fips-mode.html)
- AlmaLinux: [FIPS validation for AlmaLinux](https://almalinux.org/blog/2023-09-19-fips-validation-for-almalinux/)

### Step 2: Verify the operating system is in FIPS mode {#os-fips-check}

These tests check that the operating system runs in FIPS mode.

1. Check the kernel flags. In FIPS mode, the `crypto.fips_enabled` kernel flag is `1`; otherwise, it's `0`:

    ```shell
    sudo sysctl -a | grep fips
    ```

    The output shows that FIPS is on at the kernel level:

    ```none
    crypto.fips_enabled = 1
    crypto.fips_name = Red Hat Enterprise Linux 9 - Kernel Cryptographic API
    crypto.fips_version = 5.14.0-570.39.1.el9_6.aarch64
    ```

1. Check the kernel log for the FIPS self-tests of the algorithms:

    ```shell
    journalctl -k -o cat -g alg:
    ```

    The output shows which algorithms passed the FIPS self-tests, and which ones FIPS mode turned off:

    ```none
    alg: self-tests for pkcs1pad(rsa-generic,sha512) (pkcs1pad(rsa,sha512)) passed
    alg: self-tests for pkcs1pad(rsa-generic,sha256) (pkcs1pad(rsa,sha256)) passed
    alg: self-tests for cbc-aes-ce (cbc(aes)) passed
    alg: self-tests for jitterentropy_rng (jitterentropy_rng) passed
    alg: poly1305 (poly1305-neon) is disabled due to FIPS
    alg: xchacha12 (xchacha12-neon) is disabled due to FIPS
    alg: xchacha20 (xchacha20-neon) is disabled due to FIPS
    ```

1. Check that the whole operating system is configured for FIPS mode:

    ```shell
    sudo fips-mode-setup --check
    ```

    The output shows that FIPS mode is on:

    ```none
    FIPS mode is enabled.
    ```

### Step 3: Verify the OpenSSL is in FIPS mode {#openssl-fips-check}

1. Check that the OpenSSL FIPS provider is active:

    ```shell
    openssl list -providers | grep -A3 fips
    ```

    The output shows the version and status of the FIPS provider:

    ```none
      fips
        name: Red Hat Enterprise Linux 9 - OpenSSL FIPS Provider
        version: 3.0.7-395c1a240fbfffd8
        status: active
    ```

1. Check that OpenSSL can calculate SHA-1 hashes. SHA-1 is weak, but FIPS mode still allows it for some legacy uses. If this command fails, OpenSSL doesn't work properly:

    ```shell
    openssl sha1 /dev/null
    ```

    The output is the SHA-1 checksum of **/dev/null**:

    ```none
    SHA1(/dev/null)= da39a3ee5e6b4b0d3255bfef95601890afd80709
    ```

1. Check that OpenSSL refuses MD5 hashes. FIPS mode doesn't allow MD5, so this command fails:

    ```shell
    openssl md5 /dev/null
    ```

    The output is an error:

    ```none
    Error setting digest
    200458BAFFFF0000:error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:crypto/evp/evp_fetch.c:355:Global default library context, Algorithm (MD5 : 95), Properties ()
    200458BAFFFF0000:error:03000086:digital envelope routines:evp_md_init_internal:initialization error:crypto/evp/digest.c:272:
    ```

    Without FIPS mode, the command prints the MD5 checksum, `MD5(/dev/null)= d41d8cd98f00b204e9800998ecf8427e`.

### Step 4: Install NGINX Plus on the operating system {#nginx-plus-instll}

1. Install NGINX Plus, as described in [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}). You can install it from the [NGINX Plus repository](https://account.f5.com/myf5). Or download the **nginx-plus** package (**rpm** or **deb**) on another system, and install it manually.

1. Check that NGINX Plus is installed, and which OpenSSL library it uses:

    ```shell
    nginx -V
    ```

    The output starts like this:

    ```none
    nginx version: nginx/1.29.3 (nginx-plus-r36)
    built by gcc 11.5.0 20240719 (Red Hat 11.5.0-5) (GCC)
    built with OpenSSL 3.2.2 4 Jun 2024
    ```

1. Add a simple HTTPS website to the NGINX Plus configuration:

    ```nginx
    server {
        listen 443 ssl;

        ssl_certificate     /etc/nginx/ssl/test.crt;
        ssl_certificate_key /etc/nginx/ssl/test.key;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
    ```

    For a test, you can create a self-signed certificate:

    ```shell
    sudo mkdir -p /etc/nginx/ssl
    sudo openssl req -newkey rsa:2048 -nodes -keyout /etc/nginx/ssl/test.key \
         -x509 -days 365 -out /etc/nginx/ssl/test.crt -subj "/CN=localhost"
    ```

1. Test the configuration, and reload NGINX Plus:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

1. From a remote host, connect to NGINX Plus with `openssl s_client`, and send the request `GET /`. Replace `<NGINX_PLUS_ADDRESS>` with the IP address or host name of NGINX Plus:

    ```shell
    (echo "GET /" ; sleep 1) | openssl s_client -connect <NGINX_PLUS_ADDRESS>:443
    ```

    The output shows the negotiated cipher, in a line such as `New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384`. Then the body of the default "Welcome to nginx!" page follows.

### Step 5: Verify compliance with FIPS {#nginx-plus-fips-check}

To check which ciphers NGINX Plus accepts, connect with specific ciphers. Use a client that isn't in FIPS mode, so that it can offer them. Note these points about `openssl s_client`:

- The `-cipher` option sets only the ciphers of TLSv1.2 and earlier. So add the `-tls1_2` option. Without it, the connection uses TLSv1.3 and succeeds, whatever `-cipher` says.
- The `-cipher` option takes OpenSSL cipher names, such as `ECDHE-RSA-AES128-GCM-SHA256`. It doesn't accept standard names, such as `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`. To see both names, run `openssl ciphers -stdname`.
- If NGINX Plus refuses the cipher, the output has `sslv3 alert handshake failure` and `New, (NONE), Cipher is (NONE)`. Otherwise, it shows the cipher, for example `Cipher is ECDHE-RSA-AES128-GCM-SHA256`. The version in that line is the oldest version of the cipher, such as `SSLv3`. The `Protocol` line shows the version that the connection uses.

#### Allowed cipher: ECDHE with AES-GCM

FIPS mode allows ECDHE key exchange with AES-GCM, so this handshake succeeds in both modes:

```shell
(echo "GET /" ; sleep 1) | openssl s_client -connect <NGINX_PLUS_ADDRESS>:443 -tls1_2 -cipher ECDHE-RSA-AES128-GCM-SHA256
```

<span id="camellia-sha"></span>

#### Camellia

FIPS doesn't allow the Camellia block cipher. This test uses Camellia with ECDHE key exchange, which FIPS mode allows, so only the cipher decides. The handshake fails if NGINX Plus runs in FIPS mode, and succeeds otherwise:

```shell
(echo "GET /" ; sleep 1) | openssl s_client -connect <NGINX_PLUS_ADDRESS>:443 -tls1_2 -cipher ECDHE-RSA-CAMELLIA256-SHA384
```

<span id="aes256-sha"></span>

#### Static RSA key exchange: AES256-SHA

`AES256-SHA` uses static RSA key exchange. In FIPS mode, NGINX Plus offers no cipher suites with static RSA key exchange, as the [cipher scan](#ciphers-disabled-in-fips-mode) shows. So the handshake fails in FIPS mode, and succeeds otherwise:

```shell
(echo "GET /" ; sleep 1) | openssl s_client -connect <NGINX_PLUS_ADDRESS>:443 -tls1_2 -cipher AES256-SHA
```

The handshake fails because of the key exchange, not because of SHA-1. In FIPS mode, NGINX Plus still offers ECDHE suites with SHA-1 message authentication, such as `ECDHE-RSA-AES256-SHA`.

<span id="rc4-md5"></span><span id="3des"></span>

#### RC4 and 3DES

FIPS doesn't allow RC4, and NIST disallowed 3DES (Triple DES) encryption [after December 31, 2023](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-131Ar2.pdf). SP 800-67 also limits a 3DES key bundle to 2<sup>20</sup> data blocks.

NGINX Plus doesn't offer them anyway: its default `ssl_ciphers HIGH:!aNULL:!MD5` excludes RC4 and 3DES, as the [cipher scan without FIPS mode](#ciphers-disabled-in-fips-mode) shows. So FIPS mode has nothing to turn off here.

You also can't test these ciphers with current OpenSSL clients, which don't include them. For example, OpenSSL 3.0 on Ubuntu 24.04 fails before it connects, with an error such as `Call to SSL_CONF_cmd(-cipher, RC4-MD5) failed`.

<span id="dh-and-dsa"></span>

#### DH and DSA key sizes

NIST [SP 800-131A](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-131Ar2.pdf) requires keys of at least 2048 bits for Diffie-Hellman (DH) and the Digital Signature Algorithm (DSA). NGINX uses DHE cipher suites only with the [`ssl_dhparam`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_dhparam) directive, and neither scan shows any. For TLS, use ECDHE key exchange, as in the [allowed cipher](#allowed-cipher-ecdhe-with-aes-gcm) above.

## Ciphers disabled in FIPS mode {#ciphers-disabled-in-fips-mode}

FIPS 140-2 allows only a [subset of the usual SSL and TLS ciphers](https://csrc.nist.gov/csrc/media/publications/fips/140/2/final/documents/fips1402annexa.pdf), and [FIPS 140-3](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.140-3.pdf) has stricter requirements.

This test lists the ciphers that NGINX Plus offers to SSL/TLS clients, with the `nmap` tool, which you install separately. NGINX Plus uses its default [`ssl_ciphers HIGH:!aNULL:!MD5`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_ciphers) setting:

```shell
nmap --script ssl-enum-ciphers -p 443 <NGINX_PLUS_ADDRESS>
```

The output for NGINX Plus on RHEL 9 without FIPS mode:

```none
PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CCM (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CCM (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_ARIA_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_ARIA_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256 (rsa 2048) - A
|     compressors:
|       NULL
|     cipher preference: client
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_128_CCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     cipher preference: client
|_  least strength: A
```

The output with FIPS 140-3 mode on:

```none
PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|     compressors:
|       NULL
|     cipher preference: client
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_128_CCM_SHA256 (secp256r1) - A
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|     cipher preference: client
|_  least strength: A
```

Comparing the two scans, FIPS mode turns off these cipher suites:

- **Static RSA key exchange**: All suites whose names start with `TLS_RSA_WITH_`, with any cipher, including AES-GCM, AES-CCM, and AES-CBC. These suites also lack forward secrecy. The ones with Camellia and ARIA are in the next lists.

  - `TLS_RSA_WITH_AES_128_CBC_SHA`
  - `TLS_RSA_WITH_AES_128_CBC_SHA256`
  - `TLS_RSA_WITH_AES_128_CCM`
  - `TLS_RSA_WITH_AES_128_GCM_SHA256`
  - `TLS_RSA_WITH_AES_256_CBC_SHA`
  - `TLS_RSA_WITH_AES_256_CBC_SHA256`
  - `TLS_RSA_WITH_AES_256_CCM`
  - `TLS_RSA_WITH_AES_256_GCM_SHA384`

- **Camellia**: NIST hasn't approved Camellia, although ISO/IEC standards recognize it.

  - `TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256`
  - `TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384`
  - `TLS_RSA_WITH_CAMELLIA_128_CBC_SHA`
  - `TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256`
  - `TLS_RSA_WITH_CAMELLIA_256_CBC_SHA`
  - `TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256`

- **ARIA**: Like Camellia, ARIA isn't a NIST-approved algorithm.

  - `TLS_ECDHE_RSA_WITH_ARIA_128_GCM_SHA256`
  - `TLS_ECDHE_RSA_WITH_ARIA_256_GCM_SHA384`
  - `TLS_RSA_WITH_ARIA_128_GCM_SHA256`
  - `TLS_RSA_WITH_ARIA_256_GCM_SHA384`

- **ChaCha20-Poly1305**: NIST hasn't approved it, in TLSv1.2 or TLSv1.3.

  - `TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256`
  - `TLS_AKE_WITH_CHACHA20_POLY1305_SHA256` (TLSv1.3)

In FIPS mode, NGINX Plus still offers ECDHE key exchange with AES, in GCM mode and in CBC mode, including CBC suites with SHA-1. In TLSv1.3, it still offers AES-GCM and AES-CCM, and the key exchange changes from X25519 to the NIST curve secp256r1.

You can also check the ciphers that NGINX Plus offers with the [Qualys SSL server test](https://www.ssllabs.com/ssltest).

<span id="conclusion"></span>

## Definition of terms

- **Cryptographic module**: The OpenSSL software, with libraries of FIPS-validated algorithms that other applications can use.

- **Cryptographic boundary**: The operational functions that use FIPS-validated algorithms. For NGINX Plus, it includes all functions of the [`http_auth_jwt`](https://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html), [`http_ssl`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html), [`http_v2`](https://nginx.org/en/docs/http/ngx_http_v2_module.html), [`http_v3`](https://nginx.org/en/docs/http/ngx_http_v3_module.html), [`mail_ssl`](https://nginx.org/en/docs/mail/ngx_mail_ssl_module.html), and [`stream_ssl`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html) modules. These modules handle SSL and TLS for incoming and outgoing connections over HTTP, HTTP/2, HTTP/3, TCP, and mail protocols.

- **NGINX Plus**: The NGINX Plus software of F5, Inc., delivered in binary form from F5 servers.

- **FIPS mode**: A mode of the operating system. In it, the OpenSSL cryptographic module runs in a mode validated for FIPS 140-2 Level 1 or FIPS 140-3 Level 1. Most operating systems don't run in FIPS mode by default, so you must configure it.

- **FIPS validated**: An authorized laboratory formally validated a component of the OpenSSL cryptographic module. The validation holds if the module is built from source without changes to the source or the build process. The FIPS mode in the OpenSSL packages of operating system vendors contains this validated module.

- **FIPS compliant**: NGINX Plus is compliant with FIPS 140-2 Level 1 and FIPS 140-3 Level 1, within the cryptographic boundary. This applies when it uses a FIPS-validated OpenSSL cryptographic module, on an operating system in FIPS mode.

<span id="see-also"></span>

## References

- [FIPS 140-3 standard (PDF)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.140-3.pdf)
- [FIPS compliance with NGINX Plus and Red Hat Enterprise Linux](https://www.f5.com/pdf/technology-alliances/fips-compliance-made-simple-with-f5-and-red-hat-one-pager.pdf)
- [F5 NGINX Plus running on Red Hat Enterprise Linux is now FIPS 140-3 compliant](https://www.redhat.com/en/blog/f5-nginx-plus-running-red-hat-enterprise-linux-now-fips-140-3-compliant)

## Revision history

- Version 3 (November 2026): Fixed the cipher tests, rebuilt the list of disabled ciphers from the cipher scans, and updated the FIPS 140-2 transition.
- Version 2 (September 2025): Added information about FIPS 140-3 Level 1 compliance, updated test procedures, updated product versions, updated URLs to operating systems, NIST certificates and other relevant resources.
- Version 1 (August 2019): Initial version with FIPS 140-2 Level 1 compliance.
