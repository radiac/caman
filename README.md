caman
=====

A self-signing certificate authority manager - create your own certificate
authority, and generate and manage SSL certificates.

If you want to see how caman works, read the accompanying article,
[Self-Signing Certificate Authorities][]

This document explains how to use caman to
[create a certificate authority](#creating-a-certificate-authority), and to
[create, sign, renew and revoke](#managing-host-certificates) host
certificates.


Creating a Certificate Authority
================================

1. Clone this repository:

        git clone https://github.com/radiac/caman.git

2. Optional: Copy files into a new private repository

   The ``.gitignore`` is set up to ignore all files CAMan will create. This is
   to prevent you from accidentally pushing secrets to a public repository.
   
   If you don't want to store your certificate authority in a git repository,
   you can work inside this cloned public repository.
   
3. Configure the files in ``ca`` - (see [Configuration](#configuration))
   
4. Initialise caman in the current directory::

        ./caman init

   * You will be asked for a PEM key - this must be at least 4 characters long,
     but the longer the better. Keep it safe - you will need it whenever caman
     needs to use the CA key.

   You are now ready to create and manage host certificates.

5. Optional: Distribute ``ca/ca.crt.pem`` for your host certificates to be
   recognised.

6. Publish ``ca/ca.crl.pem`` at the URL in your configuration (unless you
   disabled CRL in your config).

Keep ``ca/ca.key.pem`` private. If it is compromised, you will need to destroy
your certificate authority and start again.


Configuration
-------------

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
* Change 4 of the values under ``[ host_distinguished_name ]``:
  * ``countryName``: the two-character country code for this host
  * ``stateOrProvinceName``: the state or province for this host
  * ``organizationName``: the name of the organisation for this host
  * ``emailAddress``: the e-mail address for the admin for this host
  * Do not change ``commonName`` or ``organizationUnitName`` - these are placeholders which will be set by caman
* The lifespan of your host certs is ``default_days``, 10 years by default


Managing host certificates
==========================

Host certificates are found in the ``store`` directory. Each host has its own
directory with the config and signing request, and each sign operation creates
a new directory with today's date. Use the files inside the latest directory.

To add a new host::

    caman new <hostname> [oun]

* ``oun`` is the organisational unit name - you'll probably want to use the
  hostname again. If the argument is missing, you will be prompted for it.

To create a new CSR, private key and signed certificate::

    caman sign <hostname>

To revoke a certificate::

    caman revoke <hostname>

* Whenever you revoke a certificate, you will need to re-publish
  ``ca/ca.crl.pem``

To replace an expired or compromised host certificate (revokes then signs)::

    caman renew <hostname>
