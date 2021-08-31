.. _doc_krill_hsm:

Integrating with an HSM
=======================

.. versionadded:: vT.B.D

Krill uses a "signer" to create and manage keys and to sign data with them. By default Krill uses an OpenSSL based
signer on the host system where Krill runs. If you have access to a hardware security module (HSM) you can instead
delegate creation, ownership, management of and signing with keys to the HSM.

.. Note:: One-off signing keys (used in MFT/ROA EE certificates) will **NOT** be created with, stored in or signed
          with the HSM. This is because it can be slow to generate, sign with and destroy one-off signing keys
          using an HSM yet one-off signing keys do not need to be protected to the same degree as RPKI CA private
          keys, parent/child identity keys or CA/publication server identity keys.

.. Warning:: The RPKI RFCs do not define a way to roll identity keys (used for parent/child and CA/publication server
             trust relationships). While Krill supports regeneration of identity keys, they cannot be rolled in an
             automated way as none of the parent NIR/RIR currently support doing so. The only way to change an
             identity key is to re-do the XML exchanges involved in establishing trust.

Compatible HSMs
---------------

In theory Krill supports any HSM that is compatible with the PKCS#11 and/or KMIP 1.2 standards. The HSM must
already be setup and you must already be in possession of any access credentials which Krill will need to use to
connect to the HSM.

Krill has been tested with the following (in alphabetical order):

  - AWS Cloud HSM (PKCS#11)c
  - Kryptus Cloud HSM v1.25.0 (KMIP)
  - PyKMIP v0.10.0 (KMIP)
  - SoftHSMv2 2.6.1 (PKCS#11)
  - YubiHSM2 (PKCS#11)

Krill can use a cluster of HSMs if the cluster appears to Krill as a single HSM, i.e. if Krill is not aware that
the "single" HSM is in fact a cluster of HSMs.

Scenarios
---------

Fresh installation
""""""""""""""""""

With a fresh installation of Krill you can use the HSM from the start. No keys will be stored locally, instead
all keys will be stored in the HSM.

Migrating to or between HSMs
""""""""""""""""""""""""""""

Krill does not support migration of existing RPKI CA private keys from one signer to another. Instead you will need
to perform a key roll for each CA. **NOTE:** See the warning above about migration of parent/child and CA/publication
server relationships to new identity keys.

Configuration
-------------

See ``krill.conf`` for full details.

.. Note:: Any changes to the configuration file will not take effect until Krill is restarted.

No explicit configuration is required to use the default OpenSSL signer. To use a signer other than the default you
must add one or more ``[[signers]]`` sections to your ``krill.conf`` file, one for each signer that you wish to define.

All signers must have a ``type`` and a ``name`` and one or more properties specific to that signer type. The ``name``
is the unique identifier for the signer and should not be changed after keys have been created otherwise Krill will
no longer know which signer to contact when trying to use a key that is "owned" by that signer.

When configuring more than one signer, one may be designated the "default" signer and another (or the same one) may
be designated the "keyroll" signer. The default signer is used to create all new keys, except in the case of keyroll
in which the specified keyroll signer will be used to create the new key.

The default configuration is equivalent to add the following in ``krill.conf``: _(using the actual ``data_dir`` path
instead of the placeholder in the example below)_

.. code-block::

   [[signers]]
   type = "OpenSSL"
   name = "OpenSSL"
   keys_path = "$data_dir/keys"
   default = true
   keyroll = true

Configuring a PKCS#11 signer
""""""""""""""""""""""""""""

For a PKCS#11 signer you must specify the path to the dynamic library file for the HSM that was supplied by the HSM
provider and a "user_pin". You can also optionally supply a "slot_id". For example:

.. code-block::

   [[signers]]
   type = "PKCS#11"
   name = "SoftHSMv2 via PKCS#11"
   lib_path = "/usr/local/lib/softhsm/libsofthsm2.so"
   user_pin = "xxxx"
   slot_id = 0x12a9f8f7
   default = true

Configuring a KMIP signer
"""""""""""""""""""""""""

For a KMIP signer you must specify the host name and optionally other connection details such as port number, client
certificate, server CA certificate, username and password.

.. code-block::

   [[signers]]
   type = "KMIP"
   name = "Kryptus via KMIP"
   host = "my.hsm.example.com"
   port = 5696
   insecure = false
   server_ca_cert_path = "/path/to/some/ca.pem"
   client_cert_path = "/path/to/some/cert.pem"
   client_cert_private_key_path = "/path/to/some/key.pem"
   username = "user1"
   password = "xxxxxx"

Usage
-----

TODO:

- Testing that Krill is able to use the HSM without restarting Krill.
- Monitoring the health of the connection to the HSM via Prometheus metrics.
- Identifying the signer that owns a particular key.

FUTURE:

- Automated rolling of identity keys? *(pending a defined way to do so)*.
- Signing of Krill audit logs?