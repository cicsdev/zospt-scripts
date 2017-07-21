# Remove Old Containers

Find and remove any `zospt` containers older than _x_
days. A [perl script](../src/find-old-containers) can return `zospt` containers older
than _x_ days. A [shell script](../src/remove-old-containers) is provided to collect all
existing containers, find the old ones, and remove them.

Read the [full blog post][blog] for further detail, and the JCL to use to schedule the
running of these scripts.

[blog]: https://developer.ibm.com/cics/2017/07/21/scripting-zos-provisioning-toolkit-removing-old-containers/
