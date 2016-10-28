.. _configuration:

=============
Configuration
=============

.. _install-area:

-----------------------------------
Basic settings in ``config.yaml``
-----------------------------------

Spack's basic configuration options are set in ``config.yaml``.  You can
see the default settings by looking at
``etc/spack/defaults/config.yaml``:

.. literalinclude:: ../../../etc/spack/defaults/config.yaml
   :language: yaml

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Config file variables
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You may notice some variables prefixed with ``$`` in the settings above.
Spack understands several variables that can be used in values of
configuration parameters.  They are:

  * ``$spack``: path to the prefix of this spack installation
  * ``$tempdir``: default system temporary directory (e.g, from the
    ``$TMPDIR`` environment variable)
  * ``$user``: name of the current user

Note that, as with shell variables, you can write these as ``$varname``
or with braces to distinguish the variable from surrounding characters:
``${varname}``

^^^^^^^^^^^^^^^^^^^^
``install_tree``
^^^^^^^^^^^^^^^^^^^^

Controls where Spack will install packages.  Default is
``$spack/opt/spack``.

^^^^^^^^^^^^^^^^^^^^
``module_roots``
^^^^^^^^^^^^^^^^^^^^

Controls where Spack installs modules it generates.  You can customize
the location for each type of module.  e.g.:

.. code-block:: yaml

   module_roots:
     tcl:    $spack/share/spack/modules
     lmod:   $spack/share/spack/lmod
     dotkit: $spack/share/spack/dotkit

^^^^^^^^^^^^^^^^^^^
``build_stage``
^^^^^^^^^^^^^^^^^^^

Spack is designed to run out of a user home directories, and on many
systems the home directory is NFS mounted *not* a fast filesystem.  We
create build stages in a temporary directory to avoid this.  Many systems
impose quotas on home directories, and ``/tmp`` or similar directories
often have more available space.

By default, the ``build_stage`` is set like this:

.. code-block:: yaml

   build_stage:
    - $tempdir
    - /nfs/tmp2/$user
    - $spack/var/spack/stage

This is an ordered list of paths that Spack should search when trying to
find a temporary directory for the build stage.  The list is searched in
order, and Spack will use the first directory to which it has write access.

When Spack creates a temporary build stage, it places a symbolic link to
the stage in ``$spack/var/spack/stage`` so that it can track the stage
and clean it up later.  By default, build stages are removed after a
successful package build.  You can manually clean build stages with
:ref:`spack purge <cmd-spack-purge>`.

Note that the last item in the list is ``$spack/var/spack/stage``.  If
this is used as the build stage, Spack will build *directly* in its
prefix and will not use temporary space.

^^^^^^^^^^^^^^^^^^^^
``source_cache``
^^^^^^^^^^^^^^^^^^^^

Location to cache downloaded tarballs and repositories.  By default these
are stored in ``$spack/var/spack/cache``.  These are stored indefinitely
by default. See :ref:`spack purge <cmd-spack-purge>`.

^^^^^^^^^^^^^^^^^^^^
``misc_cache``
^^^^^^^^^^^^^^^^^^^^

Temporary directory to store long-lived cache files, such as indices of
packages available in repositories.  Defaults to ``~/.spack/cache``.


^^^^^^^^^^^^^^^^^^^^
``verify_ssl``
^^^^^^^^^^^^^^^^^^^^

When set to ``true`` (default) Spack will verify certificates of remote
hosts when making ``ssl`` connections.  Set to ``false`` to disable, and
tools like ``curl`` will use their ``--insecure`` options.  Disabling
this can expose you to attacks.  Use at your own risk.

^^^^^^^^^^^^^^^^^^^^
``checksum``
^^^^^^^^^^^^^^^^^^^^

When set to ``true``, Spack verifies downloaded source code using a
checksum, and will refuse to build packages that it cannot verify.  Set
to ``false`` to disable these checks.  Disabling this can expose you to
attacks.  Use at your own risk.

^^^^^^^^^^^^^^^^^^^^
``clean``
^^^^^^^^^^^^^^^^^^^^

By default, Spack clears variables in your environment that can change
the way packages build. This includes ``LD_LIBRARY_PATH``, ``CPATH``,
``LIBRARY_PATH``, ``DYLD_LIBRARY_PATH``, and others.

You can set this to ``false`` to skip the environmnet cleaning step and
build in a "dirty" environment.  Be aware that this will reduce the
reproducibility of builds.


.. _sec-external_packages:

-----------------
External Packages
-----------------

Spack can be configured to use externally-installed
packages rather than building its own packages. This may be desirable
if machines ship with system packages, such as a customized MPI
that should be used instead of Spack building its own MPI.

External packages are configured through the ``packages.yaml`` file found
in a Spack installation's ``etc/spack/`` or a user's ``~/.spack/``
directory. Here's an example of an external configuration:

.. code-block:: yaml

   packages:
     openmpi:
       paths:
         openmpi@1.4.3%gcc@4.4.7 arch=linux-x86_64-debian7: /opt/openmpi-1.4.3
         openmpi@1.4.3%gcc@4.4.7 arch=linux-x86_64-debian7+debug: /opt/openmpi-1.4.3-debug
         openmpi@1.6.5%intel@10.1 arch=linux-x86_64-debian7: /opt/openmpi-1.6.5-intel

This example lists three installations of OpenMPI, one built with gcc,
one built with gcc and debug information, and another built with Intel.
If Spack is asked to build a package that uses one of these MPIs as a
dependency, it will use the the pre-installed OpenMPI in
the given directory. Packages.yaml can also be used to specify modules

Each ``packages.yaml`` begins with a ``packages:`` token, followed
by a list of package names.  To specify externals, add a ``paths`` or ``modules``
token under the package name, which lists externals in a
``spec: /path`` or ``spec: module-name`` format.  Each spec should be as
well-defined as reasonably possible.  If a
package lacks a spec component, such as missing a compiler or
package version, then Spack will guess the missing component based
on its most-favored packages, and it may guess incorrectly.

Each package version and compilers listed in an external should
have entries in Spack's packages and compiler configuration, even
though the package and compiler may not every be built.

The packages configuration can tell Spack to use an external location
for certain package versions, but it does not restrict Spack to using
external packages.  In the above example, if an OpenMPI 1.8.4 became
available Spack may choose to start building and linking with that version
rather than continue using the pre-installed OpenMPI versions.

To prevent this, the ``packages.yaml`` configuration also allows packages
to be flagged as non-buildable.  The previous example could be modified to
be:

.. code-block:: yaml

   packages:
     openmpi:
       paths:
         openmpi@1.4.3%gcc@4.4.7 arch=linux-x86_64-debian7: /opt/openmpi-1.4.3
         openmpi@1.4.3%gcc@4.4.7 arch=linux-x86_64-debian7+debug: /opt/openmpi-1.4.3-debug
         openmpi@1.6.5%intel@10.1 arch=linux-x86_64-debian7: /opt/openmpi-1.6.5-intel
       buildable: False

The addition of the ``buildable`` flag tells Spack that it should never build
its own version of OpenMPI, and it will instead always rely on a pre-built
OpenMPI.  Similar to ``paths``, ``buildable`` is specified as a property under
a package name.

If an external module is specified as not buildable, then Spack will load the
external module into the build environment which can be used for linking.

The ``buildable`` does not need to be paired with external packages.
It could also be used alone to forbid packages that may be
buggy or otherwise undesirable.


.. _concretization-preferences:

--------------------------
Concretization Preferences
--------------------------

Spack can be configured to prefer certain compilers, package
versions, depends_on, and variants during concretization.
The preferred configuration can be controlled via the
``~/.spack/packages.yaml`` file for user configuations, or the
``etc/spack/packages.yaml`` site configuration.

Here's an example packages.yaml file that sets preferred packages:

.. code-block:: yaml

   packages:
     opencv:
       compiler: [gcc@4.9]
       variants: +debug
     gperftools:
       version: [2.2, 2.4, 2.3]
     all:
       compiler: [gcc@4.4.7, gcc@4.6:, intel, clang, pgi]
       providers:
         mpi: [mvapich, mpich, openmpi]

At a high level, this example is specifying how packages should be
concretized.  The opencv package should prefer using gcc 4.9 and
be built with debug options.  The gperftools package should prefer version
2.2 over 2.4.  Every package on the system should prefer mvapich for
its MPI and gcc 4.4.7 (except for opencv, which overrides this by preferring gcc 4.9).
These options are used to fill in implicit defaults.  Any of them can be overwritten
on the command line if explicitly requested.

Each packages.yaml file begins with the string ``packages:`` and
package names are specified on the next level. The special string ``all``
applies settings to each package. Underneath each package name is
one or more components: ``compiler``, ``variants``, ``version``,
or ``providers``.  Each component has an ordered list of spec
``constraints``, with earlier entries in the list being preferred over
later entries.

Sometimes a package installation may have constraints that forbid
the first concretization rule, in which case Spack will use the first
legal concretization rule.  Going back to the example, if a user
requests gperftools 2.3 or later, then Spack will install version 2.4
as the 2.4 version of gperftools is preferred over 2.3.

An explicit concretization rule in the preferred section will always
take preference over unlisted concretizations.  In the above example,
xlc isn't listed in the compiler list.  Every listed compiler from
gcc to pgi will thus be preferred over the xlc compiler.

The syntax for the ``provider`` section differs slightly from other
concretization rules.  A provider lists a value that packages may
``depend_on`` (e.g, mpi) and a list of rules for fulfilling that
dependency.


.. _configuration-scopes:

-------------------------
Configuration Scopes
-------------------------

Spack pulls configuration data from files in several directories. There
are three configuration scopes.  From lowest to highest:

1. **defaults**: Stored in ``$(prefix)/etc/spack/defaults/``. These are
   the "factory" settings. Customizations should be made in the other
   higher precedence scopes, as defaults may change from version to
   version.

2. **site**: Stored in ``$(prefix)/etc/spack/``.  Settings here affect
   only *this instance* of Spack.  They override ``defaults``.  The site
   scope can can be used for per-project settings (one spack instance per
   project) or for site-wide settings on a multi-user machine (e.g., for
   a common spack instance).

3. **user**: Stored in the home directory, ``~/.spack/``. These settings
   affect all instances of Spack and take the highest precedence.

When configurations conflict, settings from higher-precedence scopes
override lower-precedence settings.

Commands that modify scopes (e.g., ``spack compilers``, ``spack repo``,
etc.) take a ``--scope=<name>`` parameter that you can use to control
which scope is modified.

.. _platform-scopes:

^^^^^^^^^^^^^^^^^^^^^^^^^^
Platform-specific scopes
^^^^^^^^^^^^^^^^^^^^^^^^^^

For each scope above, there can *also* be platform-specific settings. For
example, on Blue Gene/Q machines, Spack needs to know the location of
cross-compilers for the compute nodes.  This configuration is in
``etc/spack/defaults/bgq/compilers.yaml``.  It will take precedence over
settings in the ``defaults`` scope, but can still be overridden by
settings in ``site``, ``site/bgq``, ``user``, or ``user/bgq``. So, the
full scope precedence is:

1. ``defaults``
2. ``defaults/<platform>``
3. ``site``
4. ``site/<platform>``
5. ``user``
6. ``user/<platform>``

Each configuration directory may contain several configuration files,
such as ``config.yaml``, ``compilers.yaml``, or ``mirrors.yaml``.

^^^^^^^^^^^^^^^^^^^^^^^^^^^
Configuration file format
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Configuration files are written in YAML.  For more details on the format,
see `yaml.org <http://yaml.org>`_ and `libyaml
<http://pyyaml.org/wiki/LibYAML>`_.  Each file is structured as nested
dictionaries and lists.  e.g., ``config.yaml`` might look like this:

.. code-block:: yaml

   config:
     install_tree: $spack/opt/spack
     module_roots:
       lmod:   $spack/share/spack/lmod
     build_stage:
       - $tempdir
       - /nfs/tmp2/$user

Note that everything in the config file needs to be nested under a
top-level section corresponding to the config file's name.  So,
``config.yaml`` starts with ``config:``, and ``mirrors.yaml`` starts with
``mirrors:``.

^^^^^^^^^^^^^^^^^^^^^^^^^
Merging scopes
^^^^^^^^^^^^^^^^^^^^^^^^^

By default, Spack attempts to merge configurations across scopes.  That
is, if there are ``config.yaml`` files in both the default and site
scopes, Spack will merge the settings between the two files.  You can see
the results of this merge with the ``spack config get <configtype>``
command. For example, if your configurations look like this:

**defaults** scope:

.. code-block:: yaml

   config:
     install_tree: $spack/opt/spack
     module_roots:
       lmod:   $spack/share/spack/lmod
     build_stage:
       - $tempdir
       - /nfs/tmp2/$user

**site** scope:

.. code-block:: yaml

   config:
     install_tree: /some/other/directory

Spack will only override ``install_tree`` in the ``config`` section, and
will take the site preferences for other settings:

.. code-block:: console

   $ spack config get config
   config:
     install_tree: /some/other/directory
     module_roots:
       lmod:   $spack/share/spack/lmod
     build_stage:
       - $tempdir
       - /nfs/tmp2/$user
   $ _

Sometimes, it is useful to *completely* override lower-precedence
settings.  To do this, you can use *two* colons at the end of a key in a
configuration file.  For example, if the **site** ``config.yaml`` above
looks like this:

.. code-block:: yaml

   config::
     install_tree: /some/other/directory

Spack will ignore all lower-precedence configuration in the ``config::``
section:

.. code-block:: console

   $ spack config get config
   config:
     install_tree: /some/other/directory
