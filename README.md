# oci-fix-secrets-dir-hook

This OCI hook is a workaround for a bug in podman that occurs when using
the `--secret` option to mount a chosen secret in the container and at the
same time using either `--userns=auto`, `--uidmap`, `--gidmap`, `--subuidname`
or `--subgidname` options to run the container in a new user namespace.
It prevents mounting any secret in a container running in a user namespace.

> See https://github.com/containers/podman/issues/12779

## Details about the bug and its fix

This bug was caused by the fact that the secrets directory 
`<GRAPH_ROOT>/overlay-containers/<CONTAINER_ID>/userdata/secrets/`
needs to be world searchable so users can access it from different user
namespaces. (Daniel J Walsh @rhatdan)

This bug was handled and fixed by the podman team. The fix is present in podman
starting from version v4.0.0-rc2. Please feel free to verify by yourself.

> See [containers/podman commit 83b0fb4696fc9db304365eb16720c26bad93e474](https://github.com/containers/podman/commit/83b0fb4696fc9db304365eb16720c26bad93e474)

The problem is that, as of the time of posting, many distribution repositories
do not offer yet the official fixed podman version neither a patched version
with this fix.

The following is the current status of some repositories:

| Linux distribution                  | Podman version | Have the fix |    Workaround solves the bug   |
| ---                                 |      :---:     |    :---:     |           :---:                |
| Ubuntu 22.10 (Kinetic Kudu)         |      3.4.4     |   &cross;    | (Not yet tested / Should work) |
| Ubuntu 22.04 LTS (Jammy Jellyfish)  |      3.4.4     |   &cross;    | (Not yet tested / Should work) |
| Ubuntu 21.10 (Impish Indri)         |      3.2.1     |   &cross;    | (Not yet tested / Should work) |
| Ubuntu 21.04 (Hirsute Hippo)        |       N/A      |     N/A      |             N/A                |
| Ubuntu 20.04 LTS (Focal Fossa)      |       N/A      |     N/A      |             N/A                |
| opensuse.org kubic (Ubuntu 22.04)   |       N/A      |     N/A      |             N/A                |
| opensuse.org kubic (Ubuntu 21.10)   |      3.4.2     |   &cross;    | (Not yet tested / Should work) |
| opensuse.org kubic (Ubuntu 21.04)   |       N/A      |     N/A      |             N/A                |
| opensuse.org kubic (Ubuntu 20.04)   |      3.4.2     |   &cross;    |           &check;              |
| opensuse.org kubic (Ubuntu 18.04)   |      3.4.2     |   &cross;    | (Not yet tested / Should work) |

**Please feel free to make a pull request to add and/or update data in this table.**

## OCI hook workaround

This repository contains a workaround to fix the secrets-dir bug. It's basicly
a normal OCI hook that is executed in the `precreate` stage supported by podman.
It changes the permission of the secrets dir to allow users to access it from
different user namespaces. To summarize, it executes `chmod 755 <SECRESTS_DIR>`
every time a container is created.

This workaround was tested with some versions (listed in the previous table) but
should work with all versions that implement the discussed options and do not
include the secrets-dir fix (older that v4.0.0-rc2) and for sure they should
support OCI hooks and the `precreate` stage.

For easy setup, a [setup](setup) script is included. You must execute it as root.

```
git clone https://github.com/aminosbh/oci-fix-secrets-dir-hook.git
cd oci-fix-secrets-dir-hook
sudo ./setup
```

Then add manually the 'hooks_dir' option to podman configuration file
`/etc/containers/containers.conf` as follows:

```
[engine]              
hooks_dir=["/etc/containers/oci/hooks.d"]
```

> See 'man containers.conf' for more details.

**IMPORTANT NOTE:**  
Be careful if `/etc/containers/oci/hooks.d` dir already exists and contains
other hooks. They will be automatically considered for all containers. In such
a case please consider not adding `hooks_dir` option globally and use instead
the `--hook-dir` option with `podman run` whenever needed. You can also choose
to setup this hook manually in a different directory and still configure it globally.

> Please see [oci-hooks.5.md](https://github.com/containers/podman/blob/main/pkg/hooks/docs/oci-hooks.5.md)
> for further details about OCI hooks and Podman.

In case you encountered some problems with this workaround (It should not, but
in case) you can uninstall it by executing as root the [purge](purge) script and
undo any manual operations you did (maybe editing `/etc/containers/containers.conf`)

## License

Author: Amine B. Hassouna [@aminosbh](https://github.com/aminosbh)

This hook is distributed under the terms of the MIT license
[&lt;LICENSE&gt;](LICENSE).

