Slurm
=====

Install and configure a Slurm cluster on RHEL/CentOS or Debian/Ubuntu servers

Why this fork?
--------------

The upstream version of this Slurm role, while very useful, contained some bugs and missed some features we
needed/wanted for our Slurm cluster. Aside from fixes, the main expansions compared to upstream are as follows:

- Fixed handling of non-default cluster names in the Slurm database
- Made `become` explicit throughout the role where it is truly needed as I prefer it to be as local as possible. Note,
  however, that you still need to be able to `become`.
- Can now install slurmrestd as an additional component
- Supports additional plugin installs such as `job_container/tmpfs`

Note that there may also be downsides compared to upstream, e.g. I didn't test these changes against Debian-based
distros (yet). In fact, I'm only reasonably sure they work on AlmaLinux 9 targets as that is what we use.

Role Variables
--------------

All variables are optional. If nothing is set, the role will install the Slurm client programs, munge, and
create a `slurm.conf` with a single `localhost` node and `debug` partition.
See the [defaults](defaults/main.yml) and [example playbooks](#example-playbooks) for examples.

For the various roles a slurm node can play, you can either set group names, or add values to a list, `slurm_roles`.

- group slurmservers or `slurm_roles: ['controller']`
- group slurmexechosts or `slurm_roles: ['exec']`
- group slurmdbdservers or `slurm_roles: ['dbd']`
- group slurmrestapiservers or `slurm_roles: ['rest']`

General config options for slurm.conf go in `slurm_config`, a hash. Keys are Slurm config option names.

Partitions and nodes go in `slurm_partitions` and `slurm_nodes`, lists of hashes. The only required key in the hash is
`name`, which becomes the `PartitionName` or `NodeName` for that line. All other keys/values are placed on to the line
of that partition or node.

Options for the additional configuration files [acct_gather.conf](https://slurm.schedmd.com/acct_gather.conf.html),
[cgroup.conf](https://slurm.schedmd.com/cgroup.conf.html) and [gres.conf](https://slurm.schedmd.com/gres.conf.html)
may be specified in the `slurm_acct_gather_config`, `slurm_cgroup_config` (both of them hashes) and
`slurm_gres_config` (list of hashes) respectively.

Additional plugins to be installed go in `slurm_plugins`, lists of hashes. Required keys are

- `name`, the name of the plugin.
- `pkgs`, a list containing the names of the plugin's required packages, to be installed via the system's package
  manager. Make sure your package manager will be able to find them, i.e. provide additional repos beforehand if
  necessary. Leave empty for built-in plugins or if the install is handled by other means.
- `targets`, a list containing the node classes the plugin should be installed on. Use the same role specifiers as in
  `slurm_roles`, i.e. `controller`, `exec`, `dbd` or `rest`.

Optional keys for each plugin are

- `slurm_conf_entries`, a hash containing additional (!), plugin-related key (option) value pairs for `slurm.conf`,
  following the same structure as `slurm_config`. Note that these options are merged into `slurm_config` before
  templating and on all nodes, independent of selected `targets`.
- `plugin_conf_name`, name of an additional config file for the plugin. Placed in slurm's config directory (alongside
  `slurm.conf`).
- `plugin_conf_template`, template for the additional config file. Defaults to `plugin.conf.j2` if not provided. The
  default works for config files of the same structure as `slurm.conf`. Note: Ignored completely if `plugin_conf_name`
  isn't also provided.
- `plugin_conf_entries`, a hash containing key value pairs for the plugin's additional config file. Note: Ignored
  completely if `plugin_conf_name` isn't also provided.

Set `slurm_upgrade` to true to upgrade the installed Slurm packages.

You can use `slurm_user` (a hash) and `slurm_create_user` (a bool) to pre-create a Slurm user so that uids match.

Dependencies
------------

None.

Example Playbooks
-----------------

Minimal setup, all services on one node:

```yaml
- name: Slurm all in One
  hosts: all
  vars:
    slurm_roles: ['controller', 'exec', 'dbd']
  roles:
    - role: galaxyproject.slurm
      become: True
```

More extensive example:

```yaml
- name: Slurm execution hosts
  hosts: all
  roles:
    - role: galaxyproject.slurm
      become: True
  vars:
    slurm_cgroup_config:
      CgroupMountpoint: "/sys/fs/cgroup"
      CgroupAutomount: yes
      ConstrainCores: yes
      TaskAffinity: no
      ConstrainRAMSpace: yes
      ConstrainSwapSpace: no
      ConstrainDevices: no
      AllowedRamSpace: 100
      AllowedSwapSpace: 0
      MaxRAMPercent: 100
      MaxSwapPercent: 100
      MinRAMSpace: 30
    slurm_config:
      AccountingStorageType: "accounting_storage/none"
      ClusterName: cluster
      GresTypes: gpu
      JobAcctGatherType: "jobacct_gather/none"
      MpiDefault: none
      ProctrackType: "proctrack/cgroup"
      ReturnToService: 1
      SchedulerType: "sched/backfill"
      SelectType: "select/cons_tres"
      SelectTypeParameters: "CR_Core"
      SlurmctldHost: "slurmctl"
      SlurmctldLogFile: "/var/log/slurm/slurmctld.log"
      SlurmctldPidFile: "/var/run/slurmctld.pid"
      SlurmdLogFile: "/var/log/slurm/slurmd.log"
      SlurmdPidFile: "/var/run/slurmd.pid"
      SlurmdSpoolDir: "/var/spool/slurmd"
      StateSaveLocation: "/var/spool/slurmctld"
      SwitchType: "switch/none"
      TaskPlugin: "task/affinity,task/cgroup"
      TaskPluginParam: Sched
    slurm_create_user: yes
    slurm_gres_config:
      - File: /dev/nvidia[0-3]
        Name: gpu
        NodeName: gpu[01-10]
        Type: tesla
    slurm_munge_key: "../../../munge.key"
    slurm_nodes:
      - name: "gpu[01-10]"
        CoresPerSocket: 18
        Gres: "gpu:tesla:4"
        Sockets: 2
        ThreadsPerCore: 2
    slurm_partitions:
      - name: gpu
        Default: YES
        MaxTime: UNLIMITED
        Nodes: "gpu[01-10]"
    slurm_roles: ['exec']
    slurm_user:
      comment: "Slurm Workload Manager"
      gid: 888
      group: slurm
      home: "/var/lib/slurm"
      name: slurm
      shell: "/usr/sbin/nologin"
      uid: 888
```

License
-------

MIT

Author Information
------------------

Original Authors:

- [Nate Coraor](https://github.com/natefoo)
- [Helena Rasche](https://github.com/erasche)

[View contributors on GitHub](https://github.com/galaxyproject/ansible-slurm/graphs/contributors)

This fork:

- [Alexander Birk](https://github.com/alibi1125)
