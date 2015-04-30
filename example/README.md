One-time setup
===

To set up your buildfarm member configuration:

* Install this role: `ansible-galaxy install 2ndquadrant.pgbuildfarm_member`

* Copy this `example` directory into the git repository you'll use for managing your buildfarm members. Make sure to copy the `.gitignore` too.

* Generate a password for the Ansible vault, which will be used to encrypt credentials. I suggest the command `pwgen -sn 12`. Record this securely somewhere *outside* this git repository. **You only need to do this when you set up the repository, not for every host**.

* Generate a ssh private key:

        ssh-keygen -f ansible_ssh_key -t rsa

  Set a passphrase when prompted and record it somewhere secure *outside this repository*.

* `git add` the ssh key if you intend to keep it in the repository. (Otherwise edit `ansible.cfg` to remove `private_key_file=ansible_ssh_key`).

        git add ansible_ssh_key ansible_ssh_key.pub

Per-host setup
===

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

Quick Start - Running
===

To run the playbook - installing packages, cloning and test-running the
buildfarm, registering cron jobs, etc, copy your vault password into
`.vault-pass.txt` then:

    ssh-add ansible_ssh_key
    ansible-playbook playbook.yml

Authentication
===

Ansible uses a private key in this repository by default. It is encrypted with the
vault password. Pass `-k` to ansible to be prompted for the key passphrase, or
just `ssh-add` the key first.

If you want to use another key, just `ssh-add` it.

If you get "too many authentication failures for user <x>" it's because the remote
sshd didn't like how many ssh keys you have on your keyring and thinks you're trying
to DoS it.

If you'd prefer not to put the vault password in a file, pass `--ask-vault-pass`
and remove `vault_password_file=.vault_pass.txt` from `ansible.cfg`.

If you want to use a password for sudo, you can; just supply it to Ansible via
a `host_vars` vault file for that host to set `ansible_sudo_pass='foobar'`.
Or if it's the same for all hosts, use the command line option `--ask-sudo-pass`.

Quick tips
===

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

Configuration customisation
====

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

Hosts with multiple buildfarm members
===

See `multiple-members.yml` for an example that shows use of both gcc and clang
on a host. Note that the variables referenced here are defined in each host's
`host_vars`.

The general idea is that you include the `pgbuildfarm_member` role multiple times
in a playbook's hostgroup, with each role invocation parameterised by overriding
its variables.

If you want a custom config file per-animal, override the
`pgbuildfarm_config_template` variable for each role invocation to point to a
different config file path.

Why is nothing happening?
===

Did you run in check-only (`-C`) mode?

Check `ansible.log` or run `ansible-playbook` with `-v`.

Why do tests keep re-running?
===

If the test build doesn't complete, the test run will not generate a status file.

Check the logs in buildroot/HEAD/${animalname}.lastrun-logs/ particularly configure.log

Hosts with very different configurations
===

If all your hosts are really different, consider listing them in individual
`hosts:` sections, one per host, instead of using the hostgroup. This is good
if (say) one host has four animals, another has just one, etc.

Editing vault files
===

To edit a file in a vault:

    ansible-vault edit --vault-password-file=.vault_pass.txt host_vars/example1/pgbuildfarm_secret.yml

Or use "create" to create one.

Figuring out what's going on
===

Stuff to read:

* http://docs.ansible.com/intro_configuration.html
* http://docs.ansible.com/intro_inventory.html#inventory
* http://docs.ansible.com/playbooks_intro.html
* http://docs.ansible.com/playbooks_roles.html
* http://docs.ansible.com/playbooks_variables.html
* http://docs.ansible.com/template_module.html
* http://docs.ansible.com/faq.html
* http://docs.ansible.com/playbooks_conditionals.html
* http://docs.ansible.com/YAMLSyntax.html
* http://docs.ansible.com/playbooks_vault.html
* http://docs.ansible.com/playbooks_vault.html
* http://docs.ansible.com/common_return_values.html
* http://docs.ansible.com/playbooks_best_practices.html
* http://docs.ansible.com/playbooks_error_handling.html
* http://docs.ansible.com/playbooks_delegation.html
* http://docs.ansible.com/playbooks_variables.html#passing-variables-on-the-command-line
* http://jinja.pocoo.org/docs/dev/templates/#builtin-filters
* http://codepoets.co.uk/2014/ansible-random-things/
* http://hakunin.com/six-ansible-practices
