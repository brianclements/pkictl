# PKICTL

Pkictl is a biased Public Key Infrastructure (PKI) management helper tool for
implementing an internal network PKI. It's designed to make certificate, key
creation, and management tasks formal, easy to initiate, and easy to automate.
The complexities of the openssl settings are to be absorbed by the various
configuration files, saving only execution of the tasks for the script itself.

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

**STATUS:** pkictl is v.0.10.0-beta

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
- Signing/Issuing CA: CA at the bottom of a PKI hierarchy. Issues only end
  entity certificates.

### Certificate Types

- Root Certificate: Self-signed CA certificate at the root of a PKI hierarchy.
  Serves as the PKI's trust anchor.
- CA Certificate: Certificate of a CA. Used to sign certificates and CRLs.
- End Entity/User Certificate: End-user certificate issued for one or more purposes:
  email-protection, server-auth, client-auth, code-signing, etc.  A user
  certificate cannot sign other certificates.

## Infrastructure Assumptions

This script makes assumptions about how you intend to build out your PKI. It
assumes:

1. 3-tier hierarchy
    * One root CA
    * At least one intermediate CA. Can be _n..._ parent/child
    CA's or _n..._ sibling intermediate CA's.
    * A final generation of _n..._ siblings of signing CA's.
2. 1-to-1 certificate/configuration pairing
    * One configuration file per CA
    * One configuration file per end entity CSR type.

Each generation inherits a copy of the chain from it's predecessors, so we have
an opportunity to use/extend the line of certificates, using all the same CLI
behavior and folder layout, indefinitely as needed keeping only the signing
certificates we need on any particular machine and safely storing parent CA's we
don't need elsewhere.

## Setup

Remember that `pkictl` does not do any configuring itself, it merely initiates
that which is already setup via configuration files. To use:

1. Plan out your PKI first.
    1. The three most important OpenSSL configuration settings are `keyUsage`,
       `basicConstraints`, and `extendedKeyUsage`. These are not general purpose
       and need to be specific to your PKI design in order for security policies
       to be enforced properly. This is why design must be pre-planned.
2. Modify the configuration files to match your design.
    1. Parity between the naming of the configuration files and all the openssl
       output must be preserved. This is how the script can sign certificates
       across generations and amongst sibling certificates. A strong naming
       convention is desirable anyway to help keep track of all the keys, CSR's,
       certificates etc. The name scheme for pkictl does the following:

        `<domain/org name>-[<CA subdomain labels>...].root.<certificate type>[.<artifact suffix>][.<artifact file format>]`

        where:
        
        * `<domain/org name>`: name of internal network. "Myorg.local"
        * `<CA subdomain labels>`: subordinate levels below "root". This is
          "sub", "tls.sub", "node.tls.sub", etc.
        * `<certificate type>`: "ca" for a configuration that issues certificate
          authorities, "ee" for a configuration that issues end entity
          certificates.
        * `<artifact suffix>`: determined by file type. ".csr", ".key", ".crt",
          ".crl", ".conf", etc. 
        * `<artifact file format>`: if `<artifact suffix>` is supplied only,
          assume "PEM" ASCII format. When "DER" format is used, this suffix will
          be appended to the filename.

        An example hierarchy of this naming convention could look like this:

            myorg.local-root.ca
            └── myorg.local-sub.root.ca
                ├── myorg.local-tls.sub.root.ca
                │   └── server.myorg.local-node.tls.sub.root.ee
                └── myorg.local-email.sub.root.ca
                    └── myorg.local-user.email.sub.root.ee

        Some Description:

        * `myorg.local-root.ca`: The root CA
            * `myorg.local-sub.root.ca`: An intermediate CA, child of root
                * `myorg.local-tls.sub.root.ca`: A signing CA, child of sub,
                  grandchild of root
                    * `server.myorg.local-node.tls.sub.root.ee`: An end
                      entity certificate, child of tls.sub.root.ca,
                      great-grandchild of root
                * `myorg.local-email.sub.root.ca`: A signing CA, child of sub,
                  grandchild of root
                    * `server.myorg.local-user.email.sub.root.ee`: An end
                      entity certificate, child of email.sub.root.ca,
                      great-grandchild of root

        The list of configuration files would be:

            .
            ├── myorg.local-root.ca.conf
            ├── myorg.local-sub.root.ca.conf
            ├── myorg.local-tls.sub.root.ca.conf
            ├── myorg.local-node.tls.sub.root.ee.conf
            ├── myorg.local-email.sub.root.ca.conf
            └── myorg.local-user.email.sub.root.ee.conf
            
3. Setup environment

    Configuration via the CLI is kept to an absolute minimum so that
    configuration files can store the configuration complexity, not your brain.
    However, some configuration is welcomed for certain items to allow for
    efficient use of configuration files.

    It is also possible to setup these variables into a configuration file, `pkictl`
    will automatically look for these file: `pkictl.conf`, `$XDG_CONFIG_HOME/pkictl.conf`,
    `$HOME/.pkictl.conf` and finally `/etc/pkictl/pkictl.conf`. The first file found
    is taken into acount.

    * CA Settings
        * `$PKICTL_ORG`: This sets "myorg.local" in the above examples. Must be
          the same as the "organizationName" value in the openssl configuration
          file.
        * `$PKICTL_CONFIG_DIR`: defaults to `$PWD`, sets location of
          configuration files.
        * `$PKICTL_CA_DIR`: defaults to `$PWD`, sets root location of your CA
          directory. This is where all newly generated certs and keys go.
        * `$PKICTL_CA_EXTENSIONS`: set this prior to signing commands to select
          alternate certificate extensions to include from your configuration
          file.  When unset, defaults from your configuration file are used.
        * `$PKICTL_CA_POLICY`: set this prior to signing commands to select
          alternate distinguished name matching policies for signing your
          certificate. When unset, defaults from your configuration file are
          used.
    * PKCS#12 Import Settings
        * `$PKICTL_SSL_DIR`: defaults to `/etc/ssl`, make sure this coincides
          with where your operating system's OpenSSL installation stores its
          certs/keys. It is where `pkictl eecert import` will install the
          directory containing end-entity certs and private keys. The
          organization name is automatically extracted from the certificates
          themselves and used as a subfolder under `/etc/ssl`.
        * `$PKICTL_CLIENTCERTS_DIR`: defaults to `/etc/ssl/certs`, sets
          directory for symlinking root/intermediate CA certs if your
          installation has an alternative spot for imported certs (i.e.
          `/usr/local/share/ca-certificates`). In other words, a location where
          your installation's `update-ca-certificates` command can find your
          custom certs and do it's hashing.
        * `$PKICTL_IMPORT_DIR`: defaults to `$PWD`, sets location of where to
          look for PKCS#12 files for `pkictl eecert import` command.
        * `$PKICTL_PKCS12_PASS`: password used to open the PKCS#12 file.
        * `$PKICTL_IMPORT_GROUP`: group which will have RO access to newly
          imported private key. Defaults to 'ssl-cert'.
        * `$PKICTL_IMPORT_USER`: user which will be added to group 'ssl-cert' so
          that they can access the private key folder in general. However, only
          members of the group `$PKICTL_IMPORT_GROUP` can actually access the
          private key itself as it will be in it's own subdirectory. See the
          "Import" section below for details.
    * Misc. Settings
        * `$PKICTL_SAFE_TEST`: When set to 'true', all tasks that involve sudo,
          chown, or chmod with root or other system users/groups are skipped or
          run as current user instead with a notification message to alert user
          to what was not performed. This is for testing both manually and with
          the pkictl_test script.

## Usage

The subcommands are grouped into 3 categories: `rootca`, `subca`, and `eecert`.
Below are real examples using the provided configuration files.

### Rootca

Setup a typical self-signed root certificate:

>`Usage: pkictl rootca <action>`

* `pkictl rootca init`
* `pkictl rootca request`
* `pkictl rootca sign`
* `pkictl rootca gencrl`

### Subca

Setup and manage the intermediate certificates from the first sub-root
certificate, all the way to the issuing/signing certificates at the very bottom
of the PKI hierarchy:

>`Usage: pkictl subca <action> <subca label> [<signing CA label>]`

* `pkictl subca init sub`
* `pkictl subca request sub`
* `pkictl subca sign tls.sub sub`
* `pkictl subca gencrl email.sub`
* `pkictl subca genpem sub root`
* `pkictl subca revoke sub root`

### Eecert

Issue and import end-entity certificates from the issuing/signing certificates at the
bottom of the PKI hierarchy.

>`Usage: pkictl eecert <action> (<request label>|<end entity label>|<export name>) <signing CA label> [<output label>|<export name>]`

* `pkictl eecert request node.tls.sub tls.sub somehostname.localnet`
* `pkictl eecert sign somehostname.localnet tls.sub`
* `pkictl eecert genpkcs12 somehostname.localnet tls.sub exportname.somehost.localnet`
* `pkictl eecert revoke somehostname.localnet tls.sub`
* `pkictl eecert import exportname.somehost.localnet`

#### Import

The `pkictl eecert import` sub-command takes the PKCS#12 bundle, extracts the
certs and keys contained within, and imports them into logical and typical folder/file
structures for various uses. The specific folder choices are highly configurable
via environment variables and is designed with automation in mind. 

The command will:

* create a re-usable folder structure based on the value of "organizationName"
  from the end entity certificate being imported.
* create a full chain bundle file from root to end-entity for nginx style packaging
* create a file of intermediate certificates for Apache style packaging
* split out all certs into individual files and place into your OS's `ssl/certs` directory
* extract private key
* run `update-ca-certificates` for you to hash and import them.

File permissions and ownership of private keys are a tricky, but critically
important thing. A hybrid solution of best practices have been used here to
allow for a pretty durable workflow when importing keys. A single server could
potentially have multiple certificates and private keys for various programs.
They should all be segregated by program and use-case.

 1. First, we make sure a system group named 'ssl-cert' exists.
 2. Only members of that group can even access the private key folder (at
    `/etc/ssl/Myorg/private` for example), which is 710 root:ssl-cert by
    default.
 3. We then make a sub-folder in the private folder named
    `$PKICTL_IMPORT_GROUP`. So a default folder of `private/ssl-cert` will be
    710 root:ssl-cert and the keys within it will be 640 root:ssl-cert.
 4. During subsequent import processes, if `pkictl eecert import` is run with
    env var `PKICTL_IMPORT_GROUP=my-prog`, then new subfolder is created such as
    `private/my-prog` with 710 root:my-prog and the keys within are 640
    root:my-prog.
 5. Users/groups won't be created, but will be checked before trying to chown
    things. User creation should be handle by the process that needs to act as
    that user. The script will fail if they don't exist.
 6. All imported keys will default to ssl-cert, and thus be stored in that
    folder under that group. This is a great default, but also allows other
    programs to create their own folders with their own groups. The key
    segregation visually reflects the user/group segregation.
    Groups/programs/users only have access to the keys meant for them and them
    only.
 7. A purely `private/root` folder with root:root keys can also be made to
    work as well in this usage pattern for mail and web servers that start as
    root first, then drop privileges later.
    
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

## License

MIT
