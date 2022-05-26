# ansible-role-borgmatic

An Ansible role to configure automated [borg] backups using [borgmatic], compatible with [rsync.net].

## Requirements

If remote servers are configured in [repository-restricted] mode, and key based authentication is enforced, a privileged key on the control node must be used to perform management of the remote backup servers. In order to protect repository integrity in the event that the control node is compromised, this key should be passphrase protected.

Once the privileged public key has been manually added to the remote backup servers `authorized_keys` file, password authentication can be disabled. For [rsync.net] this can be done using the [rsync.net web interface]. 

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
<td>borgmatic_user</td>
<td>root</td>
<td>username</td>
</tr>
<tr>
<td>borgmatic_local_ssh_options</td>
<td>' '</td>
<td>

```yaml
'-p 22 -i ~/.ssh/id_rsa'
```

</td>
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
<td>borgmatic_remote_options</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_remote_options:
  - remote: user@example.com
    remote_path: borg1
    append_only: true
    storage_quota: 100G
    restrict:
      repositories:
        - repo1
        - repo2
      paths:
        - path1
        - path2
```

</td>
</tr>
<tr>
<td>borgmatic_configs</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_configs:
  - name: remote

    # Optional, if set to 'absent',
    # specified borgmatic config will be
    # removed, as well as remote access
    # and any related services and timers
    # defaults to 'present'
    state: present

    # Location of borgmatic config file
    path: /etc/borgmatic.d/remote.yaml

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
<tr>
<td>borgmatic_restore</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_restore:
  # Repository to restore from
  - repository: user@example.com:/path/to/repo

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
<td>borgmatic_perform_restore</td>
<td>false</td>
<td>true</td>
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
<td>Ensure key based authentication is configured with remote backup servers hosting repositories specified in borgmatic_configs</td>
</tr>
<tr>
<td>borgmatic_configure_repositories</td>
<td>Initialize repositories specified in borgmatic_configs if they do not already exist</td>
</tr>
<tr>
<td>borgmatic_configure_services</td>
<td>Ensure systemd services and timers for backups and integrity checks specified in borgmatic_configs are configured</td>
</tr>
<tr>
<td>borgmatic_user</td>
<td>User that runs borgmatic</td>
</tr>
<tr>
<td>borgmatic_local_ssh_options</td>
<td>SSH command line options used by control node when managing keys on remote backup servers</td>
</tr>
<tr>
<td>borgmatic_remote_public_keys</td>
<td>Public keys belonging to backup servers to be registered in the known_hosts file</td>
</tr>
<tr>
<td>borgmatic_remote_options</td>
<td>Server-side options applied to authorized_keys entries on remote backup servers</td>
</tr>
<tr>
<td>borgmatic_restore</td>
<td>Defines restoration that will be performed when borgmatic_perform_restore is set to true</td>
</tr>
<tr>
<td>borgmatic_configs</td>
<td>Defines borgmatic config details</td>
</tr>
<tr>
<td>borgmatic_perform_restore</td>
<td>Perform restoration of backup as defined in borgmatic_restore</td>
</tr>
</table>

`borgmatic_remote_options`, `borgmatic_configs` and `borgmatic_restore` can include multiple items following the same structure.

For `borgmatic_configs`, if an independent `check` schedule is set, consistency checking will be decoupled from backups, otherwise consistency checks are run subsequent to each backup.

Append only mode is enforced server side, and not set at repository initialization. This allows for periodic manual pruning to save space while protecting remote integrity.

It is recommended that sensitive variables be stored in a [vault]. Variables shared between hosts can be placed in a group vault such as `group_vars/all.yml`, and host specific variables can be placed in a host specific vault such as `host_vars/inventory_hostname.yml`.

### Restoring

To restore a previous backup, set `borgmatic_restore` with restoration details:

```yaml
borgmatic_restore:
  # Repository to restore from
  - repository: user@example.com:/path/to/repo
  
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

borgmatic_local_ssh_options: '-i ~/.ssh/id_rsa'
```

## Example `host_vars/inventory_hostname.yml`

This is a starting point for a typical Linux workstation:

```yaml
# Variables for specific host <inventory_hostname>

---

borgmatic_configs:
  # Typical setup for daily backups to a remote server
  - name: remote
    state: present
    path: /etc/borgmatic.d/remote.yaml
    encryption: repokey-blake2

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

      hooks:
        healthchecks: https://hc-ping.com/your-uuid-here

  # Typical setup for occasional backups to a removable drive
  # Note that since no schedule is specified, this config will not run automatically
  # Repository initialization will fail if the removable drive is not mounted
  - name: local
    state: present
    path: /etc/borgmatic.d/local.yaml
    encryption: authenticated-blake2

    content:
      location:
        source_directories:
          - /home/username

        repositories:
          - /media/username/backup

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

      hooks:
        # Exit if external drive is not mounted
        before_backup:
          - findmnt /media/username/backup > /dev/null || exit 75

borgmatic_restore:
  - repository: user@example.com:/path/to/repo
    destination: '/'

```

[borg]: https://www.borgbackup.org/
[borgmatic]: https://torsion.org/borgmatic/
[rsync.net]: https://www.rsync.net/products/borg.html
[repository-restricted]: https://borgbackup.readthedocs.io/en/stable/usage/serve.html
[rsync.net web interface]: https://www.rsync.net/am/dashboard.html
[vault]: https://docs.ansible.com/ansible/latest/cli/ansible-vault.html
