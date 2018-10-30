## caman

A self-signing certificate authority manager - create your own certificate
authority, and generate and manage SSL certificates using openssl.

If you want to see how caman works and why it exists, you read the
accompanying article,
[Self-Signing Certificate Authorities](http://radiac.net/blog/2015/05/self-ca/)

This document explains how to use caman to
[create a certificate authority](#creating-a-certificate-authority), optionally [use an intermediate CA](#using-an-intermediate-ca), and to
[create, sign, renew and revoke](#managing-host-certificates) host
certificates.

Version 0.3.2, 2016-11-24. For changelog and upgrade information, see
[Changes](CHANGES.md)

### Quickstart

To create a certificate authority and start signing:

    git clone https://github.com/radiac/caman.git
    cd caman
    cp ca/caconfig.cnf.default ca/caconfig.cnf && vi ca/caconfig.cnf
    cp ca/host.cnf.default ca/host.cnf && vi ca/host.cnf
    ./caman init
    ./caman new host.example.com
    ./caman sign host.example.com
    ./caman renew host.example.com
    ./caman revoke host.example.com

Read on to see more details, how you can do this using an intermediate certificate authority, and how to create wildcard and SAN certificates.


### Creating a Certificate Authority

1. Make sure ``openssl`` is installed on your system before using caman:
   * Debian and Ubuntu: ``sudo apt-get install openssl``

2. Clone this repository:

        git clone https://github.com/radiac/caman.git

   The ``.gitignore`` is set up to ignore all files caman will create. This is
   to prevent you from accidentally pushing secrets to a public repository.
   
   Although these instructions assume you'll keep everything in the cloned
   directory, the ``caman`` script operates on the current working directory
   and just expects it to contain the ``ca`` directory.
   
   This means you can move/symlink the script to ``/usr/local/bin/`` to make it
   available system-wide, or move the ``ca`` directory into a separate folder
   or repository.
   
3. Configure the files in the ``ca`` directory -
   (see [Configuration](#configuration))
   
4. Initialise caman in the current directory:

        cd caman
        ./caman init

   * You will be asked for a PEM key - this must be at least 4 characters long,
     but the longer the better. Keep it safe - you will need it for most caman
     commands.
   * If you plan to use a intermediate CAs, this will be your root CA.

   You are now ready to
   [create and manage host certificates](#managing-host-certificates).

5. Optional: Create an intermediate CA to do your day-to-day signing, so you
   can keep your root CA key safe and offline. See
   [Using an intermediate CA](#using-an-intermediate-ca) for details.

6. Optional: Publish ``ca/ca.crl.pem`` at the URL in your configuration
   (or you can you disable CRL in your config).

7. Optional: Distribute ``ca/ca.crt.pem`` for your host certificates to be
   recognised; see [Distribution](#distribution) for more information

Keep ``ca/ca.key.pem`` private. If it is compromised, you will need to destroy
your certificate authority and start again.


#### Configuration

Copy the default configs:
   
    cp ca/caconfig.cnf.default ca/caconfig.cnf
    cp ca/host.cnf.default ca/host.cnf

Edit both files; look for comments starting ``# >>`` for where you need to
make changes.

Changes to make in ``ca/caconfig.cnf``:
* Change the 6 values under ``[ req_distinguished_name ]``:
  * ``countryName``: your two-character country code
  * ``stateOrProvinceName``: your state or province
  * ``organizationName``: the name of your organisation
  * ``organizationUnitName``: your department in the organisation
  * ``commonName``: the name of your organisation
  * ``emailAddress``: your e-mail address
* Change the CRL distribution points URL under ``[ usr_cert ]`` and
  ``[ v3_ca ]``:
  * ``crlDistributionPoints``: URL where you will publish your ``ca.crl.pem``
  * If you don't plan on publishing a CRL, comment these lines out, as well as
    ``crl_extensions`` and ``crlnumber`` under ``[ CA_default ]``.
* The lifespan of your CA is ``default_days`` - 100 years by default

In ``ca/host.cnf``:
* Change 5 of the values under ``[ host_distinguished_name ]``:
  * ``countryName``: the two-character country code for this host
  * ``stateOrProvinceName``: the state or province for this host
  * ``organizationName``: the name of the organisation for this host
  * ``organizationUnitName``: your department in the organisation
  * ``emailAddress``: the e-mail address for the admin for this host
  * Do not change ``commonName`` - this is a placeholder which will be set by
    caman
* The lifespan of your host certs is ``default_days`` - 10 years by default


#### Distribution

You need to distribute your ``ca/ca.crt.pem`` to clients for your host
certificates to be recognised.

To install your CA cert system-wide in Debian and Ubuntu:

    sudo cp ca/ca.crt.pem /usr/local/share/ca-certificates/my_ca_name.crt
    sudo dpkg-reconfigure ca-certificates

To install your CA cert system-wide in other Linux distros:

    cp ca/ca.crt.pem "/etc/openssl/certs/$( \
        openssl x509 -inform PEM -subject_hash -in ca/ca.crt.pem | head -1 \
    ).0"

To install your CA cert system-wide in Windows:

1. For Windows Certificate Manager to recognise your certificate, you will need
   to remove the ``.pem`` file extension and distribute the file as ``ca.crt``.
2. Open the file from your filer or Internet Explorer like a normal file;
   Windows Certificate Manager will be used automatically.
3. Click "Install certificate..." and accept all defaults

Some applications (such as Firefox and Thunderbird) have their own certificate
stores; you may need to install your root certificate in these applications
separately.


### Using an intermediate CA

When running a CA, it is best practice to use an intermediate CA. You will
publish your root CA's public certificate as normal, but can store your root
CA's private key offline and use your intermediate CA to sign host
certificates.

If your intermediate CA's private key is then compromised, you can revoke your
current intermediate CA and create a new one, without needing to re-issue your
root CA's public certificate.

A paranoid user may want to create and use their root key on a machine which is
permanently air-gapped and never connects to a network. If you don't have one
of those available, it should be sufficient to move your root CA to removable
media, kept offline in a secure location.

Caman supports multiple intermediate CAs from your root CA, and intermediate
CAs can be used to create longer chains of intermediate CAs as desired.


#### Creating an intermediate CA

Creating an intermediate CA is exactly the same as creating a root CA, but
you pass the path of your root CA to the ``init`` command, and only publish
the CRT for your root CA:

1. Follow the (standard installation)[#creating-a-certificate-authority] to
   create your root CA, including publishing its CRL and distributing its CRT.

2. Create a new caman installation for your intermediate CA:

        cd ../
        mv caman caman-root
        git clone https://github.com/radiac/caman.git caman-int
        cd caman-int

   * Your caman directory names don't need to match the ones in this example;
     they don't even need to be caman installations. The ``caman`` script
     operates on the current working directory, so if you install it
     system-wide, your root and intermediate CAs can start as folders with
     nothing but a configured ``ca`` directory. Just make sure you're in the
     right directory when you call ``caman``.

3. Configure your intermediate CA using the files in ``caman-int/ca`` - (see
   [Configuration](#configuration))

   * Make sure that your ``commonName`` is unique - it must be different to
     your root CA's common name, any other intermediate CAs you create, and it
     must not match any hosts
   * Make sure that your CRL is at a different URL to that of your other CAs.

4. Initialise your intermediate CA by passing the caman dir for your root CA as
   an argument to ``init``:

        ./caman init ca:../caman-root

   * Note that CA paths are always specified with the ``ca:`` prefix
   * You now have your root CA in ``caman-root`` and your intermediate CA in
     ``caman-int``
   * Your intermediate CA's chain file is ``caman-int/ca/ca-chain.crt.pem``

   You are now ready to
   [create and manage host certificates](#managing-host-certificates) using
   the new intermediate CA.

5. Optional: Publish ``caman-int/ca/ca.crl.pem`` at the URL in your
   intermediate CA's configuration (or you can you disable CRL in your config).

6. Optional: Move your your ``caman-root`` dir to secure offline storage

Note: you can use a caman intermediate CA to create further intermediate CAs,
should you so wish.


#### Managing host certificates with an intermediate CA

Caman's syntax for managing host certificates is the same whether or not you
are using an intermediate CA, but creating a host certificate with an
intermediate CA will also create a file called ``hostname.chained.crt.pem``
(with corresponding ``hostname.chained.keycrt.pem``), which is a combined
certificate containing your host's certificate along with the intermediate CA's
trust chain.

Some servers will want you to use these combined certificates (eg nginx's ``ssl_certificate`` directive, or Dovecot's ``ssl_cert`` setting), whereas others
will want you to use the plain host certificate and provide the chain file in
``caman-int/ca/ca-chain.crt.pem`` separately (eg Apache's
``SSLCACertificateFile`` directive).


#### Revoking an intermediate CA

Revoke an intermediate CA from your root CA by passing a CA path to ``revoke``;
instead of a hostname, use the  ``ca:`` prefix, and the path to the caman dir
for your intermediate CA:

    cd caman-root
    ./caman revoke ca:../caman-int

Your intermediate CA has now been revoked; publish the updated CRL for your
root CA, ``caman-root/ca/ca.crl.pem``.

You cannot use caman to renew intermediate CAs; you have to revoke them and
start again.


### Managing host certificates

Host certificates are found in the ``store`` directory. Each host has its own
directory with the config and signing request, and each sign operation creates
a new directory with today's date. Use the files inside the latest directory.

#### Add a new host

    ./caman new <hostname> [<alt> [<alt> ...]]

* ``<hostname>`` is the main hostname for the certificate
* Use an asterisk subdomain to generate a wildcard certificate
* Add multiple ``<alt>`` hostnames after the main hostname to create a SAN
  certificate

Examples:
* Single host: ``./caman new myserver.example.com``
* Wildcard: ``./caman new *.example.com``
* SAN: ``./caman new myserver.example.com virtual1.example.com virtual2.example.com``

This command generates a config file for this host in
``store/hostname/config.cnf``, using the defaults you configured in
``ca/host.cnf``. You can edit this file manually to customise it further (for
example, to change the organisational unit name from your default).


#### Create a new certificate

    ./caman sign <hostname>

This will generate a new private key, CSR, and signed server certificate.
For client certificates use following command:

    ./caman client_sign <hostname>

#### Revoke a certificate

    ./caman revoke <hostname>

You will need to re-publish ``ca/ca.crl.pem`` after running this command.


#### Renew a certificate

    ./caman renew <hostname>

This revokes the existing certificate, and then creates a new one,
so is suitable for replacing both expired or compromised host certificates
You will need to re-publish ``ca/ca.crl.pem`` after running this command.
For client certificates use following command:

    ./caman client_renew <hostname>

### Contributing

Contributions are welcome, preferably via pull request.

Thanks to all contributors, who are listed in [CHANGES](CHANGES.md)
