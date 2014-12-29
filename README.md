# PKICTL

Pkictl is a biased Public Key Infrastructure (PKI) management helper tool for
implementing an internal network PKI. It's designed to make certificate, key
creation, and management tasks formal, easy to initiate, and easy to automate.
The complexities of the openssl settings are to be absorbed by the various
openssl.cnf files, saving only execution of the tasks for the script itself.

My goal was to use the same configuration files, folder layout, and
implementation behavior in every machine of my internal PKI infrastructure and
use the same script for every task that entailed. The script also is designed to
absorb the complexity of the Openssl CLI and configuration options and minimize
accidental user error.

**DISCLOSURE:** I am not an openssl or security expert, but this script _is_ the
result of much research in order to get my openssl knowledge to a functional
level. Don't blindly use this script. I stick pretty close to the configuration
and layout demonstrated in [this][pki-tutorial] PKI tutorial, but it is still
worth it to read up on some of the literature yourself if you truly need a
functional understanding of it and not rely on random scripts you find scattered
around the internet (including this one!).

## PKI Overview

### Components

- Public Key Infrastructure (PKI): Security architecture where trust is conveyed
  through the signature of a trusted CA.
- Certificate Authority (CA): Entity issuing certificates and CRLs.
- Certificate: Public key and ID bound by a CA signature.
- Certificate Signing Request (CSR): Request for certification. Contains public
  key and ID to be certified.
- Certificate Revocation List (CRL): List of revoked certificates. Issued by a
  CA at regular intervals.

### CA Types

- Root CA: CA at the root of a PKI hierarchy. Issues only CA certificates.
- Intermediate/Policy/Subordinate CA: CA below the root CA but not a signing CA.
  Issues only CA certificates.
- Signing/Issuing CA: CA at the bottom of a PKI hierarchy. Issues only user
  certificates.

### Certificate Types

- CA Certificate: Certificate of a CA. Used to sign certificates and CRLs.
- Root Certificate: Self-signed CA certificate at the root of a PKI hierarchy.
  Serves as the PKI's trust anchor.
- User Certificate: End-user certificate issued for one or more purposes:
  email-protection, server-auth, client-auth, code-signing, etc.  A user
  certificate cannot sign other certificates.

## Infrastructure Assumptions

This script makes assumptions about how you intend to build out your PKI. It
assumes:

1. 3-tier hierarchy
    1. One root CA
    2. At least one intermediate CA. Can be _n..._ parent/child
    CA's or _n..._ sibling intermediate CA's.
    3. A final generation of _n..._ siblings of signing CA's.
2. 1-to-1 certificate/configuration pairing
    1. One configuration file per CA
    2. One configuration file per CSR type.

Each generation inherits a copy of the chain from it's predecessors, so we have
an opportunity to use/extend the line of certificates, using all the same CLI
behavior and folder layout, indefinitely as needed keeping only the signing
certificates we need on any particular machine and safely storing parent CA's we
don't need.

## Setup

Remember that `pkictl` does not do any configuring itself, it merely initiates
that which is already setup via configuration files. To use:

1. Plan out your PKI first.
    1. Two very important OpenSSL configuration settings are `keyUsage`,
       `basicConstraints`. These are not general purpose and need to be specific
       to your PKI design in order for security policies to be enforced
       properly. This is why design must be pre-planned.
2. Modify the configuration files to match your design.
    1. Parity between the naming of the configuration files and all the openssl
       output must be preserved. This is how the script can sign certificates
       across generations and amongst sibling certificates. The name scheme must
       follow the following:

        `{domain/org name|FQDN}-[[CA subdomain/type labels].]{heirarchy root label}.{certificate type}[.[artifact suffix]]`

        An example tree could look like this:

            myorg.local-root.ca
            └── myorg.local-sub.root.ca
                ├── myorg.local-tls.sub.root.ca
                │   └── server.myorg.local-tls.crt
                └── myorg.local-ocsp.sub.root.ca
                    └── server.myorg.local-ocsp.crt

        Some Description:

        * `myorg.local-root.ca`: The root CA
            * `myorg.local-sub.root.ca`: An intermediate CA, child of root
                * `myorg.local-tls.sub.root.ca`: A signing CA, child of sub,
                  grandchild of root
                    * `server.myorg.local-tls.crt`: A server certificate, child of
                      tls.sub.root.ca, great-grandchild of root
                * `myorg.local-ocsp.sub.root.ca`: A signing CA, child of sub,
                  grandchild of root
                    * `server.myorg.local-ocsp.crt`: A server certificate, child of
                      ocsp.sub.root.ca, great-grandchild of root

        The list of configuration files would be:

            .
            ├── myorg.local-root.ca.cnf 
            ├── myorg.local-sub.root.ca.cnf 
            ├── myorg.local-tls.sub.root.ca.cnf  
            └── myorg.local-ocsp.sub.root.ca.cnf
            
3. Tweak script or setup environment
    1. `$ORG`: This sets "myorg.local" in the example. Must be the same as
       the "organizationName" value in the openssl configuration file.
    2. `$ROOT_DIR`: defaults to `$PWD`, change to run everything in a different
       location.

## Usage

TODO

## Sources

* Official Sources
    * [OpenSSL Docs](https://www.openssl.org/docs/)
    * [PKI-Tutorial][pki-tutorial]
    * [Digital Ocean OpenSSL Essentials](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs)
    * [Github self-signed SSL certificate document](https://help.github.com/enterprise/11.10.340/admin/articles/using-self-signed-ssl-certificates)
    * [PKI Implementation for the Linux Admin](http://www.linux.com/community/blogs/133-general-linux/742528-pki-implementation-for-the-linux-admin)
* Blogs
    * [Multiple Blog Posts](https://jamielinux.com/blog/category/CA/) by
      [Jamie Nguyen](https://jamielinux.com/about/)
    * [Be Your Own CA](http://www.g-loaded.eu/2005/11/10/be-your-own-ca/) by
      [George Notaras](http://www.g-loaded.eu/author/gnotaras/)
    * [Becoming a x509 Certificate Authority](http://www.davidpashley.com/articles/becoming-a-x-509-certificate-authority/)
      by [David Pashley](http://www.davidpashley.com/)
    * [Be Your Own Certificate Authority with OpenSSL](http://www.area536.com/projects/be-your-own-certificate-authority-with-openssl/)
      by [Bas van de Wiel](http://www.area536.com/home/)
    * [Hierarchies in PKI](http://networklore.com/hierarchies-in-pki/)

[pki-tutorial]: http://pki-tutorial.readthedocs.org/en/latest/index.html
