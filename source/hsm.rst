.. _doc_krill_hsm:

Integrating with an HSM
=======================

.. versionadded:: vT.B.D

Krill uses a "signer" to create and manage keys and to sign data with them. By default Krill uses a software OpenSSL
based signer on the host system where Krill runs. If you have access to a hardware security module (HSM) you can
instead delegate creation, ownership, management of and signing with keys to the HSM.

.. Warning:: This page documents a pre-release feature in Krill. HSM support is not yet available.

.. Note:: One-off signing keys (used in MFT/ROA EE certificates) will **NOT** be created with, stored in or signed
          with the HSM. This is because it can be slow to generate, sign with and destroy one-off signing keys
          using an HSM yet one-off signing keys do not need to be protected to the same degree as RPKI CA private
          keys, parent/child identity keys or CA/publication server identity keys.

.. Warning:: The RPKI RFCs do not define a way to :ref:`roll <Key Rollover>` identity keys (used for parent/child and
             CA/publication server trust relationships). While Krill supports regeneration of identity keys, they
             cannot be rolled in an automated way as none of the parent NIR/RIR currently support doing so. The only
             way to change an identity key is to re-do the XML exchanges involved in establishing trust.

Compatible HSMs
---------------

In theory Krill supports any HSM that is compatible with the
`PKCS#11 <https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=pkcs11>`_ and/or the 
`KMIP <https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=kmip>`_ 1.2 standards. The HSM must already be
setup and you must already be in possession of any access credentials which Krill will need to use to connect to the
HSM.

Krill has been tested with the following (in alphabetical order):

  - `AWS Cloud HSM <https://aws.amazon.com/cloudhsm/>`_ (PKCS#11)
  - `Kryptus kNET HSM v1.25.0 <https://www.kryptus.com/knet/>`_ (KMIP & PKCS#11)
  - `PyKMIP v0.10.0 <https://github.com/OpenKMIP/PyKMIP>`_ (KMIP)
  - `SoftHSMv2 2.6.1 <https://github.com/opendnssec/SoftHSMv2>`_ (PKCS#11)
  - `Utimaco Security Server 4.45.3 <https://www.utimaco.com/products/categories/general-purpose-solutions/securityserver>`_ (PKCS#11)
  - `YubiHSM2 <https://www.yubico.com/products/hardware-security-module/>`_ (PKCS#11)

In order to work with Krill the HSM must support the following operations:

===================  =================
PKCS#11              KMIP
===================  =================
C_DeleteObject       Activate
C_Finalize           Create Key Pair
C_FindObjects        Destroy
C_FindObjectsFinal   Get
C_FindObjectsInit    Modify Attribute
C_GetAttributeValue  Query
C_GetInfo            Revoke
C_GetSlotInfo        Sign
C_GetSlotList        
C_GetTokenInfo       
C_Initialize         
C_Login              
C_OpenSession        
C_Sign               
C_SignInit           
===================  =================

Krill can use a cluster of HSMs if the cluster appears to Krill as a single HSM, i.e. if Krill is not aware that
the "single" HSM is in fact a cluster of HSMs.

PKCS#11 or KMIP?
""""""""""""""""

PKCS#11 and KMIP are very similar in the capabilities they provide, so much so that there are commercial offerings
that can bridge from one to the other, HSMs may offer support for both and both standards are maintained by 
`OASIS <https://www.oasis-open.org/>`_. From a Krill server operation perspective however they are very different and
each has its own PROs and CONs.

PKCS#11 works by delegating configuration, logging, administration, maintenance and upgrade of the interface with
the HSM to a library file outside of Krill that Krill loads when it runs. You therefore have to manage and monitor
this library and its logs as a separate component on the system running Krill. However, as a separate component it
can connect in any way it needs to the backend which can be local or remote, or possibly even to a cluster of
systems. Krill sees only the library, it has no way of knowing whether the backend is local or remote, singlular or
clustered. However it also has no way of controlling how long the library will block to wait for a task to complete
or how many requests it can handle at once or how many system resources it uses.

KMIP is arguably simpler to setup. With KMIP you only need to manage Krill and the HSM, there is no additional
library component to manage as with PKCS#11. Krill itself communicates directly with the HSM and so all
configuration, logging & resource usage is determined by Krill and monitoring is done by monitoring Krill itself.
Krill connects to the KMIP server via TLS encrypted TCP and thus could also potentially be routed to one of many
backend servers in a cluster, or the server could be a process running locally on the same host such as PyKMIP.

Scenarios
---------

Fresh installation
""""""""""""""""""

With a fresh installation of Krill you can use the HSM from the start. No keys will be stored locally, instead
all keys will be stored in the HSM.

Migrating to or between HSMs
""""""""""""""""""""""""""""

Krill does not support migration of existing RPKI CA private keys from one signer to another. Instead you will need
to perform a :ref:`key roll <Key Rollover>` for each CA. **NOTE:** Not all keys can be rolled. See the warning above
about migration of parent/child and CA/publication server relationships to new identity keys.

To perform a key roll from one signer to another you must first change the ``default_signer`` in ``krill.conf`` to
the new signer, and then restart Krill. After this point any new keys that are created by Krill, including the new
key resulting from a rollover, will be created in using the new ``default_signer``.

Configuration
-------------

See ``krill.conf`` for full details.

.. Note:: Any changes to the configuration file will not take effect until Krill is restarted.

For backward compatibility if no ``[signers]`` sections exist in ``krill.conf`` then Krill will use the default OpenSSL
signer for all signing related operations. To use a signer other than the default you must add one or more
``[[signers]]`` sections to your ``krill.conf`` file, one for each signer that you wish to define.

All signers must have a ``type`` and a ``name`` and properties specific to the type of signer.

The default configuration is equivalent to addding the following in ``krill.conf``:

.. code-block::

   [[signers]]
   type = "OpenSSL"
   name = "Default OpenSSL signer"

Signer Roles
""""""""""""

When configuring more than one signer, one may be designated the ``default_signer`` and another (or the same one) may
be designated the ``one_off_signer``. The ``default_signer`` is used to create all new keys, except in the case of one-off
signing for which the ``one_off_signer`` signer will be used to create a new temporary key, sign with it then destroy it.

Specifying the ``default_signer`` and ``one_off_signer`` is done by referencing the name of the signer. For example the
above is equivalent to:

.. code-block::

   default_signer = "Default OpenSSL signer"
   one_off_signer = "Default OpenSSL signer"

   [[signers]]
   type = "OpenSSL"
   name = "Default OpenSSL signer"

When only a single signer is defined it will implicitly be the ``default_signer``. When defining more than one signer
the ``default_signer`` must be set explicitly.

If the ``default_signer`` is not of type ``OpenSSL`` and is not explicitly set as the ``one_off_signer``, an OpenSSL
signer will automatically be used as the ``one_off_signer``.

Configuring a PKCS#11 signer
""""""""""""""""""""""""""""

.. Note:: To actually use a PKCS#11 based signer you must first set it up according to the vendors instructions. This
          may require creating additional configuration files outside of Krill, setting passwords, provisioning users,
          exporting shell environment variables for use by the library while running as part of the Krill process,
          creating or determining a slot ID or label, etc.

For a PKCS#11 signer you must specify the path to the dynamic library file for the HSM that was supplied by the HSM
provider and a slot ID or label, and if needed, a user pin.

.. code-block::

   [[signers]]
   type = "PKCS#11"
   name = "SoftHSMv2 via PKCS#11"
   lib_path = "/usr/local/lib/softhsm/libsofthsm2.so"
   slot = 0x12a9f8f7                                      
   user_pin = "xxxx"                                       # optional
   login = true                                            # optional, default = true

Note:
  - If using a slot label rather than ID you can supply the label using ``slot = "my label"``.
  - You can also supply an integer slot ID, e.g. ``slot = 123456``.
  - If your HSM does not require you to login you can set ``login = false``.
  - If your HSM requires you to supply a pin via an external key pad you can omit the ``user_pin`` setting.

Configuring a KMIP signer
"""""""""""""""""""""""""

.. note:: To actually use a KMIP based signer you must first set it up according to the vendors instructions. This may
          require setting up users and passwords and/or obtaining certificates in order to populate the associated
          settings in the ``krill.conf`` file.

For a KMIP signer you must specify the host FQDN or IP address, and optionally other connection details such as port number, client
certificate, server CA certificate, username and password.

.. code-block::

   [[signers]]
   type = "KMIP"
   name = "Kryptus via KMIP"
   host = "my.hsm.example.com"
   port = 5696                                             # optional, default = 5696
   server_ca_cert_path = "/path/to/some/ca.pem"            # optional
   client_cert_path = "/path/to/some/cert.pem"             # optional
   client_cert_private_key_path = "/path/to/some/key.pem"  # optional
   username = "user1"                                      # optional
   password = "xxxxxx"                                     # optional
   insecure = false                                        # optional
   force = false                                           # optional

Note:
  - ``host`` can also be an IP address.
  - ``insecure`` will disable verification of any certificate presented by the server.
  - ``force`` should only be used if the HSM fails to advertize support for a feature that Krill requires but actually
    the HSM **does** support the feature.

Signer Lifecycle
----------------

At startup Krill will announce the configured signers in its logs but will not yet attempt to connect to them. Only
once a signing related operation needs to be performed will Krill attempt to connect to the signer.

If there is a problem connecting to a signer Krill will retry, unless the problem is fatal such as the signer lacking
support for required operations. A problem with a signer will not stop Krill from running and continuing to serve the
UI and API or from executing background tasks. Thus if some keys are owned by one signer that is reachable and another
signer is not reachable, Krill will continue to operate correctly for operations involving the reachable signer.

On initial connection to a new signer Krill will create a "signer identity key" in the HSM. This serves to verify that
the signer is able to create and sign with keys and in future that the signer is the one that owns keys attributed to
it.

New keys are created by the ``default_signer`` unless they are one-off keys in which case they are created by the
``one_off_signer``. Signing with a key is handled by the signer that possesses the key.

.. Note:: Krill determines the signer that possesses a key by consulting a mapping that it keeps from key identifier
          to a Krill internal signer ID and associated metadata.
          
          On initial connection to a signer it "binds" the internal representation of the connected signer to the
          matching internal signer ID and updates the metadata about the signer. It verifies that the internal signer
          ID corresponds to the backend by verifying the existence of a previously created "signer identity key" within
          the backend and that the backend is able to correctly sign with that key.
          
          Krill is able to maintain the mapping between keys associated with a signer ID and the actual connected
          signer even if the name and server connection details in ``krill.conf`` are changed so you are free to rename
          the signer or replace the physical server by a (synchronized) spare or upgrade or change its IP address or
          the credentials used to access it and Krill will still know when connecting to it which keys it possesses.

.. Warning:: If Krill is not configured to connect to the signer that possesses a key that Krill needs to sign with,
             or is unable to connect to it using the configured settings, then Krill will be unable to sign with that
             key!

             One particular scenario to watch out for is when reconfiguring an existing Krill instance to use an HSM
             when that Krill instance already has at least one CA (and thus already created at least one key pair
             using OpenSSL).

             In this scenario, if the changes to ``krill.conf`` to use the HSM define only the one signer (the HSM)
             and do NOT set that signer as the ``one_off_signer``, then Krill will activate the default OpenSSL signer
             for one-off key signing and will use it to find the previously created OpenSSL keys.
             
             If however the one and only HSM signer is also set as the ``one_off_signer`` then Krill will not activate
             the OpenSSL signer and so will not find the previously created OpenSSL keys. In this case you must
             explicitly add a ``[[signers]]`` block of ``type = "OpenSSL"`` with default settings thereby causing Krill
             to activate the default OpenSSL signer.

SoftHSMv2 Example
-----------------

Lets see how to setup `SoftHSMv2 <https://github.com/opendnssec/SoftHSMv2>`_ with Krill. This example uses commands
suitable for an Ubuntu operating system, for other operating systems you may need to use slightly different commands.

First, install and setup SoftHSM v2:

.. code-block::
   
   $ sudo apt install -y softhsm2
   $ softhsm2-util --init-token --slot 0 --label "My token 1" --so-pin 1234 --pin 5678

Next add the following to your `krill.conf` file:

.. code-block::

   [[signers]]
   type = "PKCS#11"
   name = "SoftHSMv2"
   lib_path = "/usr/lib/softhsm/libsofthsm2.so"
   slot = "My token 1"
   user_pin = 5678

Now (re)start Krill.

That's it! When you next create a CA Krill will create a key pair for it in SoftHSMv2 instead of using OpenSSL.

One way to inspect the keys stored inside OpenSSL is using the ``pkcs11-tool`` command:

.. code-block::

   $ sudo apt install -y opensc
   $ pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -O -p 5678
   Using slot 0 with a present token (0x542bc831)
   Public Key Object; RSA 2048 bits
     label:      Krill
     ID:         e83e96883ee73e69e0e57d54b6726c9d45f788c5
     Usage:      verify
     Access:     local
   Public Key Object; RSA 2048 bits
     label:      Krill
     ID:         9ecd3796786c7a073d5384c155d8d475d103df74
     Usage:      verify
     Access:     local
   ...
