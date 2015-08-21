## caman changes

### Changelog

**0.2.0**, 2015-08-21: Add SAN support
* See [Upgrading](#upgrading) below for upgrade instructions
* ``new`` command changed to replace OUN with alt hostnames to support SANs
* Version number now displayed when called with missing or incorrect arguments

**0.1.0**, 2015-05-16: Initial release


<a name="upgrading"></a>
### Upgrading

#### Upgrading from 0.1.0

The file ``ca/caconfig.cnf.default`` has been changed, so you need to update
your customised ``caconfig/host.cnf``:
* The section ``[ CA_default ]`` has a new setting ``copy_extensions = copy``


The file ``ca/host.cnf.default`` has been changed, and your customised
``ca/host.cnf`` needs to be changed too:
* The ``organizationalUnitName`` now has to be set manually in this file;
  change ``<<OUN>>`` to an appropriate value for your certificates. This can
  be changed manually on a per-host basis once you have run the ``new`` command.
* In the section ``[ v3_req ]``, after the line ``keyUsage``, add the line
  ``<<ALT_HOSTNAMES>>``

The changes to ``ca/host.cnf`` do not affect the configuration for existing
hosts.

* The ``new`` command no longer accepts an OUN; this now has to be set manually
  in your ``ca/host.cnf``, and customised per-host as necessary after running
  ``new``.
