---
layout: post
title:  "Self-signed CA and cert"
date:   2016-10-11 11:59:48
categories: openssl ca certificate jenkins nexus
---

**TL;DR** A script to generate a self-signed CA and server certificates

<img style="float: left;" src="/images/openssl.jpeg">

I really like internal-only wildcard certs for the cloud; you have a fleet of servers visible only to your devops team and one certificate can provide secure communications to them all. Wildcard cert costs are trivial for corporations - about $200 for the last one I bought. 

But what if you are doing this on-the-cheap? Maybe this is a personal project, or perhaps a new POC that can't wait for a request to make it through corporate channels. Or, perhaps, you need this for Consul/Vault or secure Docker communications and don't want certificate bloat to increase maintenance. All of these, and perhaps more, are good reasons to create your own Certificate Authority for generating in-house certificates.

There are many tutorials on this topic already but this took a bit of research to get just right. The bash script described can generate a CA certificate, and wildcard or dedicated server certificates, that work with Consul and Docker as well as CI/CD pipeline components like Jenkins and Nexus.

## Certificate owner information

Since there is a lot of information already available on the topic, I will only briefly introduce the owner information so we can use the included script and make a sane looking certificate. 

Owner information varies by country. For the United States:

- **C**: Country Name: a 2 letter country code, e.g. US
- **ST**: State must be spelled out, not abbreviated, as in Colorado
- **L**: Locality: in the US, this is usually the city name
- **O**: Organization: The publicly recognized company or organization name
- **OU**: Organizational Unit: the name of the group owning the certificate, such as “IT Operations” or “XYZ Web Hosting”. We are omitting it, assuming this is sufficiently apparent by the common name for CA and certificate.
- **CN**: Common Name may be a wildcard or fully qualified host name (FQHN).   

Note: We do not enter an email address or organizational unit because I don't find them of any use in this circumstance. No challenge password is used because this is for services and we don't want to answer a password challenge on server restart.

## The script

This example will use the following information

```shell
C  : US
ST : Colorado
L  : Denver 
O  : Makara Design Group 
CN : PlatformCA for CA, *.internal.mdg.com for a wildcard cert
```

These values are entered into the following bash script as shown in the first variable definitions.

Create a file named `create-ca-and-cert.sh` with the contents below in its own directory.

```shell
#!/bin/bash

# Create a self-signed root certificate (CA) and then generate a cert from that CA.
# This was specifically developed for Consul, but has also been used with the Nexus
# docker registry and Jenkins.
# NOTE: The Docker daemon interprets .crt files as CA certificates and .cert
# files as client certificates.

# Define specifics for this CA
CA_NAME=platform_ca
CA_CERT_SUBJ='/C=US/ST=Colorado/L=Denver/O=Makara Design Group/CN=PlatformCA'
CERT_NAME=splat.internal.mdg.com
CERT_CSR_SUBJ='/C=US/ST=Colorado/L=Denver/O=Makara Design Group/CN=*.internal.mdg.com'

# Clean up old entries - omit this if you will be creating several certs from the same CA
rm *.csr *.pem *.crt *.cert *.key certindex* serial* *.conf

# Serial holds the next available cert serial number and is updated after each cert gen
echo "000a" > serial

# The CA records the certificates it signs here
touch certindex

# Create a self-signed root certificate
# NOTE: -nodes eliminates a passphrase; for infrastructure nodes
# NOTE: omit this line if you will be using the same CA to sign more certificates
eval "openssl req \
      -newkey rsa:2048 -keyout ${CA_NAME}.key \
      -x509 -days 3650 -nodes -out ${CA_NAME}.crt \
      -subj \"$CA_CERT_SUBJ\""

# Create a wildcard certificate signing request
eval "openssl req \
      -newkey rsa:1024 -keyout ${CERT_NAME}.key \
      -nodes -out ${CERT_NAME}.csr \
      -subj \"$CERT_CSR_SUBJ\""

# create an openssl.cfg for this CA used when signing the CSR
cat > $CA_NAME.conf <<__EOF__
[ ca ]
default_ca = $CA_NAME

[ $CA_NAME ]
unique_subject = no
new_certs_dir = .
certificate = ${CA_NAME}.crt
database = certindex
private_key = ${CA_NAME}.key
serial = serial
default_days = 3650
default_md = sha1
policy = ${CA_NAME}_policy
x509_extensions = ${CA_NAME}_extensions

[ ${CA_NAME}_policy ]
commonName = supplied
stateOrProvinceName = supplied
countryName = supplied
emailAddress = optional
organizationName = supplied
organizationalUnitName = optional

[ ${CA_NAME}_extensions ]
basicConstraints = CA:false
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth,clientAuth

__EOF__

# Sign the generated certificates with the new CA. 
# This will produce a file called CERT_NAME.cert in the current directory.
# It will also create new versions of the serial and certindex files, 
# moving the old versions to backup files. A .pem file will also be 
# created based on the serial number in the serial file - identical to CERT_NAME.cert
# NOTE: Batch mode automatically certifies any CSRs passed in, without prompting.
eval "openssl ca -batch -config ${CA_NAME}.conf -notext -in ${CERT_NAME}.csr -out ${CERT_NAME}.cert"

# Create a .pem file for use with SSL off-loading on the host server (HAProxy)
cat ${CERT_NAME}.cert ${CERT_NAME}.key > ${CERT_NAME}.pem

```

Run it with:

```shell
$ chmod 766 create-ca-and-cert.sh
$ ./create-ca-and-cert.sh
```

## What to do with the files

First, archive this directory in secure storage. You will need this state if you generate/sign more certificates. In many cases, you will want to use a single CA to limit how many certs users have to install, and how many certs you have to install in infrastructure.

Now, let's take a look at the files that were generated.

```shell
-rw-r--r--   1 starver  staff  1159 Mar 15 17:45 0A.pem
-rw-r--r--   1 starver  staff    81 Mar 15 17:45 certindex
-rw-r--r--   1 starver  staff    20 Mar 15 17:45 certindex.attr
-rw-r--r--   1 starver  staff     0 Mar 15 17:45 certindex.old
-rwxrw-rw-   1 starver  staff  2692 Mar 16 14:46 create-ca-and-cert.sh
-rw-r--r--   1 starver  staff   685 Mar 15 17:45 platform_ca.conf
-rw-r--r--   1 starver  staff  1493 Mar 15 17:45 platform_ca.crt
-rw-r--r--   1 starver  staff  1679 Mar 15 17:45 platform_ca.key
-rw-r--r--   1 starver  staff     3 Mar 15 17:45 serial
-rw-r--r--   1 starver  staff     5 Mar 15 17:45 serial.old
-rw-r--r--   1 starver  staff  1159 Mar 15 17:45 splat.internal.mdg.com.cert
-rw-r--r--   1 starver  staff   651 Mar 15 17:45 splat.internal.mdg.com.csr
-rw-r--r--   1 starver  staff   887 Mar 15 17:45 splat.internal.mdg.com.key
-rw-r--r--   1 starver  staff  2046 Mar 15 17:45 splat.internal.mdg.com.pem
```
The two most important files are the server certificate `splat.internal.mdg.com.pem` and the CA certificat `platform_ca.crt`; they will be used frequently.

### The server certificate

The `splat.internal.mdg.com.pem` file is the concatenation of `splat.internal.mdg.com.cert` and `splat.internal.mdg.com.key`, in that order. It is installed on host servers exposing a TLS (https) service like a docker registry, a Nexus, or a Jenkins, webapp. Note that some infrastructure wants separate key and cert files - check the doc to see which is most appropriate.

On each infrastructure component that provides a TLS (https) service, I install HAProxy for SSL off-loading. A more robust service might have a redundant pair of HAProxys accessed by VIP, but the concept is the same. You will copy the pem file to, perhaps, `/etc/haproxy/certs` and reference it in the `haproxy.cfg` like this:

```shell
frontend nexus
    bind *:443 ssl crt /etc/haproxy/certs/splat.internal.mdg.com.pem no-sslv3
```

### The CA certificate

The `platform_ca.crt` file is our root CA cert which guarantees the authenticity of the server cert. It will need to be installed in each server's CA repository that is a client of a server using the certificate `splat.internal.mdg.com.pem`. This includes user's laptops as well as internal infrastructure.

For a Ubuntu 16 server, install the CA cert following these steps:

```shell
# Create a separate directory for our certs to make them easy to find
mkdir /usr/share/ca-certificates/extra

# Copy the CA .crt file to this directory - must end in crt:
cp platform_ca.crt /usr/share/ca-certificates/extra/platform_ca.crt

# Let Ubuntu add the .crt file's path relative to /usr/share/ca-certificates to /etc/ca-certificates.conf:
dpkg-reconfigure ca-certificates
```

Users connecting to a service, e.g. Nexus or Jenkins, that use the new server certificate will visit the site and get warnings that it is not trusted. You can distribute the `platform_ca.crt` to these users so they can include it in their trusted CAs.

On a mac, you can double-click the `platform_ca.crt` file and add it to your keychain. Then open KeyChain Access, find the PlatformCA entry in System Roots, double click the entry, and change "When using this certificate" to "Always Trust". 

Afterwards, Chrome will still show the site as "Not Secure" in the URL window but will load the site without going through the warning page. In FireFox, you will still have to add a security exception - follow the instructions when visiting the site. In Safari, you will see no indications of an invalid certificate.

NOTE: you will have to restart the Docker service to pick up these changes; whether on your laptop or an infrastructure component.

## Epilog

I find that I keep coming back to this process because I change groups or have new POCs, so I decided to make it easy and convert tribal knowledge to a reusable script. Hopefully, it can make your life a little easier too.
