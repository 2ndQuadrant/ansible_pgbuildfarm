pgbuildfarm_member
=========

`pgbuildfarm_member` is an [Ansible](http://www.ansible.com/) [role](https://docs.ansible.com/playbooks_roles.html) for managing a [PostgreSQL buildfarm](http://buildfarm.postgresql.org/) member ("animal").

If you have an unusual platform or configuration and want to make sure it's
well covered by PostgreSQL's tests, this role will make it easier for you to
create a buildfarm member to automatically run tests on every PostgreSQL
project git commit and report the results to the development team. Performance
regression tests are planned for the future, too.

Please *do not* contact the PostgreSQL buildfarm administrators about this
Ansible role. This isn't an official part of the buildfarm tools. See "Author
Information" below for contact details.

An example Ansible playbook and configuration is provided [in this role's
`example/` directory](https://github.com/2ndQuadrant/ansible_pgbuildfarm/tree/master/example).
It is strongly recommended that you use it, or at least
review it, as it shows things like how to use the vault to store buildfarm
member secrets, how to fix Ansible's SSH locale handling, etc.

Requirements
------------

Tested on Ansible 1.8 at time of writing.

No non-core modules required.

You *must* use a custom `ssh_config` or edit your `/etc/ssh/ssh_config`
(your `~/.ssh/config` will *not* work) to work around
[Ansible bug 10698](https://github.com/ansible/ansible/issues/10698).
If you don't then buildfarm member setup may fail.

You must disable `requiretty` in `sudoers` for the Ansible user.

If you want to report results to the PostgreSQL buildfarm you need to register
there. You can use this role for private non-reporting members or testing
first.

Role Variables
--------------

### Required variables

* `pgbuildfarm_animal`: buildfarm animal name. If you haven't registered with the buildfarm yet, make up a temporary name.

* `pgbuildfarm_secret`: buildfarm secret. Only required if `pgbuildfarm_send_enabled` is true.

### Important settings

* `pgbuildfarm_user`: System user to create/use for the buildfarm. Default `pgbuildfarm`. Created if it doesn't exist.

* `pgbuildfarm_send_enabled`: boolean, default 'false'. Enable if you want to submit results to the buildfarm. `pgbuildfarm_secret` must be set if this is true.

### Buildfarm config file template options

These options control things in the generated `build-farm.conf` from the
template.

* `pgbuildfarm_config_template`: Use a custom config file instead of the default templated file. Default is `build-farm.conf.j2` which will find the role's template file. Any other file is still run through the jina2 template engine so `{{...}}` and `{% ... %}` expressions are signficant.

* `pgbuildfarm_branches`: Perl expression for the branch(es) to build on each run. Defaults to `"'ALL'"`. You can use a Perl `qw(...)` expression here.

* `pgbuildfarm_use_vpath`: do vpath builds of PostgreSQL

* `pgbuildfarm_config_opts`: Options to pass to `./configure` for the build(s). See `defaults/main.yml` for the default, which is the same as the pgbuildfarm client example config file.

* `pgbuildfarm_config_env`: A hash/dictionary of environment variables to set when running PostgreSQL's `configure` script. Use this to set the compiler and flags if needed, enable ccache, etc. Env vars must contain embedded single quotes. e.g.:

            pgbuildfarm_config_env:
                CC: "'ccache gcc'"

    or for LLVM's clang:

            pgbuildfarm_config_env:
                CC: "'ccache clang -Qunused-arguments'"


* `pgbuildfarm_base_port`: integer, port number above 1024. Must be unique between buildfarm animals on the same host. You only need to set this if you have more than one buildfarm animal on a single host.

* `pgbuildfarm_make_jobs`: `make -j` jobcount for builds, speeds up builds on big machines with more cores.

* `pgbuildfarm_build_env`: hash/dictionary of environment vars to run `make` etc with. You usually won't need to set this.

### Settings for buildfarm execution frequency

These are settings for the `/etc/crontab` entry the role installs for a buildfarm member. Default is every 5 minutes.

* `pgbuildfarm_cron_minute`: `"*/5"`
* `pgbuildfarm_cron_hour`: `"*"`
* `pgbuildfarm_cron_day`: `"*"`
* `pgbuildfarm_cron_month`: `"*"`
* `pgbuildfarm_cron_weekday`: `"*"`

* `pgbuildfarm_cron_mailto`:  Who to send mail to. Defaults to `pgbuildfarm` user on localhost; you can reroute with `/etc/aliases` or by changing this. A local mailer daemon must be configured for this to work.

### Other role settings

* `pgbuildfarm_home`: Home directory for `pgbuildfarm_user`. Default is `/home/{{pgbuildfarm_user}}` which is right almost all the time.

* `pgbuildfarm_client`: Where to check out the pgbuildfarm client when cloning from git. Defaults to `{{pgbuildfarm_home}}/pgbuildfarm-client`.

* `pgbuildfarm_buildroot`: Where to put the git checkout of PostgreSQL, git mirror, build trees, etc. Defaults to `{{pgbuildfarm_home}}/buildroot-{{pgbuildfarm_animal}}`.

* `pgbuildfarm_config`: Where to put the config file for the buildfarm. Defaults to `{{pgbuildfarm_home}}/build-farm-{{pgbuildfarm_animal}}.conf"`.

* `pgbuildfarm_enabled`: Can be used to turn the role off. boolean, default `true`.

* `pgbuildfarm_force_test`: Even if a prior test run was successful, re-run the buildfarm test. Mostly useful from the command line. Default `false`.

* `pgbuildfarm_skip_test`: Don't test-run the buildfarm client during the play. Mostly useful from the command line. Default `false`.

* `pgbuildfarm_test_branch`: When doing a test-run, what branch to build. Doesn't affect normal buildfarm runs. Default will be a recent stable branch; see `defaults/main.yml`

* `pgbuildfarm_perl_path`: Where to find `perl`. Defaults to `/usr/bin/perl`.


Troubleshooting
---------------

### Why do tests keep re-running?

If the test build doesn't complete, the test run will not generate a status file.

Check the logs in buildroot/HEAD/${animalname}.lastrun-logs/ particularly configure.log


Dependencies
------------

None, but see "Role Variables" above for *required* variables.

Example Playbook
----------------

See `example/pgbuildfarm_members.yml` and `example/multiple_members.yml`. Here's the simplest one:

    - hosts: pgbuildfarm
      roles:
        - role: pgbuildfarm_member

Usage
-----

### One-time setup

To set up your buildfarm member configuration:

* Install this role: `ansible-galaxy install 2ndquadrant.pgbuildfarm_member`

* Copy this `example` directory into the git repository you'll use for managing your buildfarm members. Make sure to copy the `.gitignore` too.

* Generate a password for the Ansible vault, which will be used to encrypt credentials. I suggest the command `pwgen -sn 12`. Record this securely somewhere *outside* this git repository. **You only need to do this when you set up the repository, not for every host**.

* Generate a ssh private key:

        ssh-keygen -f ansible_ssh_key -t rsa

  Set a passphrase when prompted and record it somewhere secure *outside this repository*.

* `git add` the ssh key if you intend to keep it in the repository. (Otherwise edit `ansible.cfg` to remove `private_key_file=ansible_ssh_key`).

        git add ansible_ssh_key ansible_ssh_key.pub

### Per-host setup

* Install your `ansible_ssh_key.pub` in `.ssh/authorized_keys` for the host into a user who has passwordless `sudo` rights. I recommend creating an `ansible` user for this purpose.

   on RH/Fedora:

        # redhat/Fedora
        usermod -a -G adm,wheel ansible

   is enough for passwordless sudo.

   On Debian/Ubuntu, use `sudo` instead of `wheel` in the above and in `visudo` change:

        %sudo   ALL=NOPASSWD: ALL

   to

        %sudo   ALL=(ALL) NOPASSWD: ALL

   so you can `sudo -u` to any user, not just root.

For vaulted files, see "Editing vault files" below.

* Edit `/etc/ssh/sshd_config` on each host and add a line for the user you'll use for Ansible:

        Defaults:ansible !requiretty

* Edit the `hosts` file in this repo and replace the entries there with your buildfarm members. You can use short, convenient names here, it doesn't have to be a valid IP or domain name. I recommend using the short hostname. Do not use the buildfarm animal name unless that's the hostname too.

* For each listed host, create the directory `host_vars/<hostname>`

* For each listed host, copy `host_vars/example1/main.yml` to `host_vars/<hostname>/main.yml`

* Edit the copied `host_vars` `main.yml` file and set:

        ansible_ssh_host: MY_IP_OR_HOSTNAME
        ansible_ssh_port: 22
        ansible_ssh_user: ansible

  (adjusting as necessary to connect to your host).

* In the same `main.yml` file, declare your buildfarm member name(s). If you only have one buildfarm member per host it's just:

         pgbuildfarm_animal: elephant-example

  *If you don't have a buildfarm member name because you haven't registered the member yet, use a dummy name for now and change it once you've registered the animal.*

* Set the buildfarm animal's secret(s) using the Ansible vault:

        ansible-vault edit --vault-password-file=.vault_pass.txt host_vars/example1/pgbuildfarm_secret.yml

* If there are any custom options you want for the buildfarm animal, just override variables exposed by the `pgbuildfarm_member` Ansible role (see `pgbuildfarm_member/defaults/main.yml` for available settings) in your `host_vars`, `group_vars`, inventory, or per-role in the playbook.

See below for how to use a totally custom buildfarm config file, run multiple buildfarm members per host, etc.

### Running

To run the playbook - installing packages, cloning and test-running the
buildfarm, registering cron jobs, etc, copy your vault password into
`.vault-pass.txt` then:

    ssh-add ansible_ssh_key
    ansible-playbook playbook.yml

### Quick tips

append `-C` if you want check-only mode, which says what Ansible would do
without doing it.

Append `-v` for more detail including command stdout/stderr, or `-vvvv` for
tons of debug output form Ansible.

append `--limit=host` if you want just one host, e.g. `--limit=example1`. This must
be the hostname used in `hosts`.

The vault password is recorded in the remote-dba repository. You can
`--ask-vault-pass` instead of putting it in a password file if you prefer.

If you want to suppress the buildfarm test runs, use `--extra-vars "pgbuildfarm_skip_test=true"`

Special SSH configuration can go in `ansible_ssh_config`. A custom config file
is used because of locale issues in Ansible; see issue https://github.com/ansible/ansible/issues/10698

### Configuration customisation

If you want to override settings not currently parameterised via the
`pgbuildfarm_member` role's templating of `build-farm.conf`, please add support
for it by modifying `pgbuildfarm_member/defaults/main.yml` to add the
variable(s) and `pgbuildfarm_members/templates/build-farm.conf.j2` to
substitute them. See existing examples and the Ansible/jina2 docs for guidance.
Then send a pull request on github.

Alternately, you can completely replace the configuration generated by the
`pgbuildfarm_member` role by creating
`host_files/<hostname>/templates/build-farm.conf.j2` with your buildfarm
configuration. You can use the same `{{variable}}` includes as in in
`pgbuildfarm_member/templates/build-farm.conf.j2` or just hardcode everything.

### Hosts with multiple buildfarm members

See `multiple-members.yml` for an example that shows use of both gcc and clang
on a host. Note that the variables referenced here are defined in each host's
`host_vars`.

The general idea is that you include the `pgbuildfarm_member` role multiple times
in a playbook's hostgroup, with each role invocation parameterised by overriding
its variables.

If you want a custom config file per-animal, override the
`pgbuildfarm_config_template` variable for each role invocation to point to a
different config file path.

### Hosts with very different configurations

If all your hosts are really different, consider listing them in individual
`hosts:` sections, one per host, instead of using the hostgroup. This is good
if (say) one host has four animals, another has just one, etc.

### Editing vault files

To edit a file in a vault:

    ansible-vault edit --vault-password-file=.vault_pass.txt host_vars/example1/pgbuildfarm_secret.yml

Or use "create" to create one.

License
-------

The PostgreSQL license (effectively BSD)

Author Information
------------------

Craig Ringer <craig.ringer@2ndquadrant.com>
