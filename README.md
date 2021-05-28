Ansible Role: Lmod
==================

An Ansible role that installs [Lmod][] from source.

This role was written with convenient **installation on HPC clusters** in mind.
This means that it is possible to install the actual Lmod software into a
global, networked file system share on only a single host, while all other
hosts install just the Lmod dependencies and the [shell
configuration](#shell-configuration) files. Nevertheless, it is of course
possible to install Lmod with this role on a single server.

Also, it is possible with this role to incrementally [transition from another
module system](#incremental-roll-out) to Lmod.

Table of Contents
-----------------

<!-- toc -->

- [Requirements](#requirements)
- [Role Variables](#role-variables)
  * [Base Variables](#base-variables)
  * [System Spider Cache](#system-spider-cache)
  * [Shell Configuration](#shell-configuration)
  * [Incremental Roll-Out](#incremental-roll-out)
  * [Administrative Messages](#administrative-messages)
- [Dependencies](#dependencies)
- [Example Playbook](#example-playbook)
  * [Top-Level Playbook](#top-level-playbook)
  * [Role Dependency](#role-dependency)
- [License](#license)
- [Author Information](#author-information)

<!-- tocstop -->

Requirements
------------

- Ansible 2.4
- **cron** for regular [system spider cache update](#system-spider-cache)

Role Variables
--------------

### Base Variables

```yml
lmod_version: '7.7.14'
lmod_install: no
lmod_prefix: '/opt/apps'
lmod_module_root_path: '{{ lmod_prefix }}/modulefiles'
```

**For HPC clusters:** The variable `lmod_install` should only be set to `yes`
for the host that actually installs Lmod to the global, networked file system,
all other hosts will only install the dependencies. This should be set in
`host_vars`, see [examples](#example-playbook).

You can also specify the module file paths which will be taken over the default
`lmod_module_root_path`:

```yml
lmod_module_paths:
  - '/software/modulefiles/compiler'
  - '/software/modulefiles/libraries'
  - '/software/modulefiles/tools'
```

This may be useful if you have **legacy modules**.

### Site Package

You can modify the `SitePackage.lua`:

```yml
lmod_site_package: path/to/my/templates/SitePackage.lua
```

### System Spider Cache

Location of the system spider cache files:

```yml
lmod_spider_cache_dir: '{{ lmod_module_root_path }}/.spider-cache'
lmod_spider_cache_stamp_file: '{{ lmod_module_root_path }}/.spider-cache.stamp'
```

Whether to install the update system spider cache cron job:

```yml
lmod_spider_cache_cron: no
lmod_spider_cache_cron_minute: '0'
```

Change the paths to include when generating the system spider cache:

```yml
lmod_spider_cache_cron_modulepath:
  - '$MODULEPATH'
```

**Note:** It may be useful to append paths here other than the default
`MODULEPATH`, especially for communities within your system that provide their
own module directories. Suppose your default is `/software/modules`, which
everyone gets, but you also have `/software/community/group-{a,b,c}/modules`,
which can be included on demand. By adding these paths to the system spider
cache, it will include those modules *transparently*, i.e. you would only be
able to find them with `module spider`, if you had included that module
directory in your `MODULEPATH`.

**For HPC clusters:** This is also needed only on a single host, following the
same rules as `lmod_install`, see [examples](#example-playbook).

### Shell Configuration

Names of the shell configuration files which will be written to
`/etc/profile.d`:

```yml
lmod_init_bash_file: 'z00-lmod.sh'
lmod_init_csh_file: 'z00-lmod.csh'
```

Changing these is only required if they need to be `source`'d in a specific
order, e.g. if another script requires the `module` or `ml` functions, they
need to be `source`'d *after* the Lmod files.

### Incremental Roll-Out

This module provides a way for *incremental roll-out* aka *canary release*.
This may be important if you are migrating from another environment module
system to Lmod.

**Note:** For more information about canary releases in general, see [this blog
post][canary]. The Lmod documentation this specific incremental roll-out is
based on can be found [here][lmod-canary].

```yml
lmod_canary: no
```

This variable has three possible values:

1.  `no`: do not use incremental roll-out, every user will init Lmod
1.  `'opt-in'`: installs custom scripts that inits Lmod only if *opt-in* file
    exists
1.  `'opt-in-skel'`: same as *opt-in* plus also installs automatically opt-in
    new users via [skeleton](http://www.linfo.org/etc_skel.html) in `/etc/skel`
1.  `'opt-out'`: installs custom scripts that don't init Lmod if *opt-out* file
    exists

**Note:** You can switch from **opt-in** to **opt-in-skel**, from both
**opt-in** variants to **opt-out** and from **opt-out** to **no**, the tasks
provided by this role will handle these transitions automatically.

These are the file names for **opt-in** and **opt-out** that are looked for in
the users home directory:

```yml
lmod_canary_opt_in_file: '.lmod-yes'
lmod_canary_opt_out_file: '.lmod-no'
```

**Note:** If you are using the incremental roll-out, you simply need to let
your users know that they need to `touch ~/.lmod-yes` for **opt-in**, `touch
~/.lmod-no` for **opt-out** and to remove these files if they want to fallback
to the respective default behavior.

### Administrative Messages

Optionally, you can define administrative messages for modules to be shown when
they are loaded:

```yml
lmod_admin_messages:

  - pattern: gcc/2%.95
    message: >-
      This module is deprecated and will be removed from the system on Jan 1
      1999. Please switch to a newer compiler.

  - pattern: /opt/apps/modulefiles/Compiler/gcc/4.7.2/boost/1.55.0
    message: We are having issues.

  - pattern: boost/1%.[5-7].*
    message: We are having more issues.
```

Dependencies
------------

This role *conditionally* depends on [geerlingguy.repo-epel][repo-epel] for
**RedHat**-based distributions to install runtime and build dependencies. Not
all of these dependencies are included in default repositories.

Example Playbook
----------------

Add to `requirements.yml`:

```yml
---

# optional
# - src: geerlingguy.repo-epel

- src: idiv-biodiversity.lmod

...
```

Download:

```console
$ ansible-galaxy install -r requirements.yml
```

**For HPC clusters:** As explained above, some variables need to be set only on
the master/head host (whatever you call it). You should set these in the
respective `host_vars` file, e.g. `host_vars/head.yml`:

```yml
---

lmod_install: yes
lmod_spider_cache_cron: yes

...
```

### Top-Level Playbook

Write a top-level playbook:

```yml
---

- name: head server
  hosts: head
  roles:
    - role: idiv-biodiversity.lmod
      tags:
        - lmod
        - modules
  vars:
    lmod_prefix: '/software'
    lmod_install: yes
    lmod_canary: 'opt-in'

...
```

### Role Dependency

Define the role dependency in `meta/main.yml`:

```yml
---

dependencies:

  - role: idiv-biodiversity.lmod
    vars:
      lmod_prefix: '/software'
      lmod_canary: 'opt-in'
    tags:
      - lmod
      - modules

...
```

License
-------

MIT

Author Information
------------------

This role was created in 2017 by [Christian Krause][author] aka [wookietreiber
at GitHub][wookietreiber], HPC cluster systems administrator at the [German
Centre for Integrative Biodiversity Research (iDiv)][idiv]. Ubuntu Bionic support
was added by [Tom Schoonjans][tschoonj], Research Software Engineer for HPC and Cloud
at the [Rosalind Franklin Institute][rfi].


[author]: https://www.idiv.de/en/groups_and_people/employees/details/61.html
[canary]: https://martinfowler.com/bliki/CanaryRelease.html
[lmod-canary]: http://lmod.readthedocs.io/en/latest/045_transition.html
[epel]: https://fedoraproject.org/wiki/EPEL
[idiv]: https://www.idiv.de/
[Lmod]: http://lmod.readthedocs.io/en/latest/
[repo-epel]: https://galaxy.ansible.com/geerlingguy/repo-epel/
[wookietreiber]: https://github.com/wookietreiber
[tschoonj]: https://github.com/tschoonj
[rfi]: https://www.rfi.ac.uk/
