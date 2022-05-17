# ansible-role-borgmatic

An ansible role to configure automated [borg] backups using [borgmatic], compatible with [rsync.net].

## Requirements

`jmespath` must be installed on the control node.

Hosts are configured to connect to remote backup servers using [repository-restricted] mode. If key based authentication is enforced, an unrestricted key must be used to perform management tasks on the remote backup servers. In order to protect remote repository integrity in the event that the control node is compromised, this key should be passphrase protected.

If using [rsync.net], once the management public key has been manually added to the backup servers `authorized_keys` file, password authentication can be disabled through the [rsync.net web interface]. 

## Role Variables

<table>
<tr>
<th>Variable</th>
<th>Default</th>
<th>Example</th>
</tr>
<tr>
<td>borgmatic_configure_package</td>
<td>true</td>
<td>false</td>
</tr>
<tr>
<td>borgmatic_configure_ssh</td>
<td>true</td>
<td>false</td>
</tr>
<tr>
<td>borgmatic_configure_repositories</td>
<td>true</td>
<td>false</td>
</tr>
<tr>
<td>borgmatic_configure_services</td>
<td>true</td>
<td>false</td>
</tr>
<tr>
<td>borgmatic_perform_restore</td>
<td>false</td>
<td>true</td>
</tr>
<tr>
<td>borgmatic_user</td>
<td>root</td>
<td>username</td>
</tr>
<tr>
<td>borgmatic_remote_public_keys</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_remote_public_keys: 
  - example.com ssh-ed25519 AAAA...
  - example.com ssh-rsa AAAA...
  - example.com ecdsa-sha2-nistp256 AAAA...
```

</td>
</tr>
<tr>
<td>borgmatic_control_ssh_options</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_control_ssh_options:
  - '-p 22'
  - '-i ~/.ssh/id_rsa'
```

</td>
</tr>
<tr>
<td>borgmatic_restore</td>
<td></td>
<td>

```yaml
borgmatic_restore:
  # Repository to restore from
  repository: user@example.com:/path/to/repo

  # Optionally specify the archive,
  # defaults to 'latest'
  archive: latest

  # Optionally specify a list of source paths 
  # from within the archive to restore
  # defaults to entire archive
  paths:
    - /home/username
    - /etc

  # Destination that will be restored into
  destination: '/'
```

</td>
</tr>
<tr>
<td>borgmatic_configs</td>
<td></td>
<td>

```yaml
borgmatic_configs:
  - name: remote

    # Location of borgmatic config file
    path: /etc/borgmatic.d/remote.yaml

    # Optional, if set to 'absent',
    # specified borgmatic config will be
    # removed, as well as remote access
    # and any related services and timers
    # defaults to 'present'
    state: present

    # Optional, set append only mode
    # defaults to false
    append_only: true

    # Optional, set storage quota
    # defaults to no quota
    storage_quota: 500G

    # Needed for repository initialization
    encryption: repokey-blake2

    # Optional, if no schedule is set,
    # borgmatic config will be installed,
    # but no systemd services or timers
    # will be created
    schedule:
      # Optional, create systemd service
      # and timer for backups
      backup:
        # Set systemd timer 'OnCalendar'
        oncalendar: daily
        # Optional, set start delay
        delay: 5m

      # Optional, create separate systemd
      # service and timer for running
      # consistency checks
      # If not specified, checks are run
      # immediately after backups
      check:
        # Set systemd timer 'OnCalendar'
        oncalendar: weekly
        # Optional, set start delay
        delay: 1h

    # Content of borgmatic config file
    # See https://torsion.org/borgmatic/
    config:
      location:
        source_directories:
          - /home/username

        repositories:
          - user@example.com:/path/to/repo

        exclude_patterns:
          - /home/username/.cache

        # rsync.net requires borg1
        remote_path: borg1
      storage:
        encryption_passphrase: redacted

      retention:
        keep_hourly: 24
        keep_daily: 7
        keep_weekly: 4
        keep_monthly: 6
        keep_yearly: 1

      consistency:
        checks:
          - repository
          - archives
```

</td>
</tr>
</table>

### Variable Notes

<table>
<tr>
<th>Variable</th>
<th>Notes</th>
</tr>
<tr>
<td>borgmatic_configure_package</td>
<td>Ensure the borgmatic package is installed</td>
</tr>
<tr>
<td>borgmatic_configure_ssh</td>
<td>Ensure key based authentication is configured with backup servers hosting remote repositories specified in borgmatic_configs</td>
</tr>
<tr>
<td>borgmatic_configure_repositories</td>
<td>Initialize repositories specified in borgmatic_configs if they do not already exist</td>
</tr>
<tr>
<td>borgmatic_configure_services</td>
<td>Ensure systemd services and timers for backups and checks specified in borgmatic_configs are configured</td>
</tr>
<tr>
<td>borgmatic_perform_restore</td>
<td>Perform restoration of backup as defined in borgmatic_restore</td>
</tr>
<tr>
<td>borgmatic_user</td>
<td>User that runs borgmatic</td>
</tr>
<tr>
<td>borgmatic_remote_public_keys</td>
<td>Public keys belonging to backup servers to register in the known_hosts file</td>
</tr>
<tr>
<td>borgmatic_control_ssh_options</td>
<td>SSH command line options used by control node when installing keys</td>
</tr>
<tr>
<td>borgmatic_restore</td>
<td>Arguments to use when executing a restoration</td>
</tr>
<tr>
<td>borgmatic_configs</td>
<td>Multiple configurations can be specified as separate list items</td>
</tr>
</table>

<!-- If a backup schedule is set for a config, but no check schedule is set, consistency checks will be performed alongside backups. To save time and resources, an independent check schedule can be set. -->
If an independent `check` schedule is set, consistency checking will be decoupled from backups, otherwise checks are run subsequent to each backup, which can take quite a while for a large repository.

Append only mode is enforced server side, and not set at repository initialization. This allows for periodic manual pruning to save space while protecting remote integrity.

It is recommended that sensitive variables be stored in a [vault]. Variables shared between hosts can be placed in a group vault such as `group_vars/all.yml`, and host specific variables can be placed in a host specific vault such as `host_vars/inventory_hostname.yml`.

### Restoring

To restore a previous backup, set `borgmatic_restore` with restoration details:

```yaml
borgmatic_restore:
  # Repository to restore from
  repository: user@example.com:/path/to/repo

  # Optionally specify the archive,
  # defaults to 'latest'
  archive: latest

  # Optionally specify a list of source paths 
  # from within the archive to restore
  # defaults to entire archive
  paths:
    - /home/username
    - /etc

  # Destination that will be restored into
  destination: '/'
```

To perform the restoration, set `borgmatic_perform_restore=true` at playbook runtime:

```sh
ansible-playbook playbook.yml -e 'borgmatic_perform_restore=true'
```

## Example Playbook

```yaml
# Configure automated borg backups using borgmatic

---

- name: Configure borgmatic backups on workstations
  hosts: workstations
  roles:
    - role: interdependence.borgmatic
```

## Example `group_vars/all.yml`

```yaml
# Variables shared by all hosts

---

borgmatic_remote_public_keys: 
  - example.com ssh-ed25519 AAAA...
  - example.com ssh-rsa AAAA...
  - example.com ecdsa-sha2-nistp256 AAAA...

borgmatic_control_ssh_options:
  - '-p 22'
  - '-i ~/.ssh/id_rsa'
```

## Example `host_vars/inventory_hostname.yml`

This is a starting point for a typical Linux workstation:

```yaml
# Variables for specific host <inventory_hostname>

---

borgmatic_restore:
  repository: user@example.com:/path/to/repo
  destination: '/'

borgmatic_configs:
  # Typical setup for daily backups to a remote server
  - name: remote
    path: /etc/borgmatic.d/remote.yaml
    append_only: true

    schedule:
      backup:
        oncalendar: daily
        delay: 5m
      check:
        oncalendar: weekly
        delay: 1h

    content:
      location:
        source_directories:
          - /home/username

        repositories:
          - user@example.com:/path/to/repo

        exclude_patterns:
          - /home/username/.cache

        # rsync.net requires borg1
        remote_path: borg1
      storage:
        encryption_passphrase: secretpassphrase

      retention:
        keep_hourly: 24
        keep_daily: 7
        keep_weekly: 4
        keep_monthly: 6
        keep_yearly: 1

      consistency:
        checks:
          - repository
          - archives

      hooks:
        healthchecks: https://hc-ping.com/your-uuid-here

  # Typical setup for occasional backups to a removable drive
  # Note that since no schedule is specified, this config will not run automatically
  # Repository initialization will fail if the removable drive is not mounted
  - name: local
    path: /etc/borgmatic.d/local.yaml
    encryption: authenticated-blake2

    content:
      location:
        source_directories:
          - /home/username

        repositories:
          - /media/username/backup

      storage:
        encryption_passphrase: secretpassphrase

      retention:
        keep_hourly: 24
        keep_daily: 7
        keep_weekly: 4
        keep_monthly: 6
        keep_yearly: 1

      consistency:
        checks:
          - repository
          - archives

      hooks:
        # Exit if external drive is not mounted
        before_backup:
          - findmnt /media/username/backup > /dev/null || exit 75
```

[borg]: https://www.borgbackup.org/
[borgmatic]: https://torsion.org/borgmatic/
[rsync.net]: https://www.rsync.net/products/borg.html
[repository-restricted]: https://borgbackup.readthedocs.io/en/stable/usage/serve.html
[rsync.net web interface]: https://www.rsync.net/am/dashboard.html
[vault]: https://docs.ansible.com/ansible/latest/cli/ansible-vault.html
