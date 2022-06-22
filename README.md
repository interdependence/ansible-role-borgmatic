# ansible-role-borgmatic

An Ansible role to configure automated [borg] backups using [borgmatic], compatible with [rsync.net].

## Requirements

If remote backup servers are configured in [repository-restricted] mode, and key based authentication is enforced, an unrestricted key on the control node must be used to perform management of the remote backup servers. In order to protect repository integrity in the event that the control node is compromised, this key should be passphrase protected.

Once the unrestricted public key has been manually added to the remote backup servers `authorized_keys` file, password authentication can be disabled. For [rsync.net] this can be done using the [rsync.net web interface]. 

## Role Variables

<table>
<tr>
<th>Variable</th>
<th>Default</th>
<th>Example</th>
</tr>
<tr>
<td>borgmatic_install</td>
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
<td>borgmatic_configs</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_configs:
  # Location of borgmatic config file
  - path: /etc/borgmatic.d/remote.yaml

    # Optional, if set to absent,
    # specified borgmatic config will be
    # removed, as well as remote access
    # defaults to present
    state: present

    # Needed for repository initialization
    encryption: repokey-blake2

    # Content of borgmatic config file
    # See https://torsion.org/borgmatic/
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
```

</td>
</tr>
<tr>
<td>borgmatic_services</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_services:
  # Name of systemd service
  - name: borgmatic-remote-backup

    # Defaults to present
    state: present
 
    # Path of config file
    config: /etc/borgmatic.d/remote.yaml
 
    # Actions to perform
    actions:
      - prune
      - compact
      - create
 
    # Optional, create systemd timer
    timer:
      oncalendar: daily
 
  - name: borgmatic-remote-check
    state: present
    config: /etc/borgmatic.d/remote.yaml
    actions:
      - check
    timer:
      oncalendar: monthly
      randomizeddelaysec: 2h
```

</td>
</tr>
<tr>
<td>borgmatic_remote_options</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_remote_options:
  # Host specific options enforced by remote server
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
<td>borgmatic_extract</td>
<td>[ ]</td>
<td>

```yaml
borgmatic_extract:
  # Repository to extract from
  - repository: user@example.com:/path/to/repo

    # Optional, specify the archive,
    # defaults to 'latest'
    archive: latest

    # Optional, specify a list of source paths
    # from within the archive to extract
    # defaults to entire archive
    paths:
      - /home/username
      - /etc

    # Destination that will be extracted into
    destination: '/'
```

</td>
</tr>
<tr>
<td>borgmatic_perform_extract</td>
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
<td>borgmatic_install</td>
<td>Ensure the borgmatic package is installed</td>
</tr>
<tr>
<td>borgmatic_configure_ssh</td>
<td>Ensure key based authentication is configured with all remote backup servers hosting repositories specified in borgmatic_configs</td>
</tr>
<tr>
<td>borgmatic_configure_repositories</td>
<td>Ensure repositories specified in borgmatic_configs are initialized, if they do not already exist</td>
</tr>
<tr>
<td>borgmatic_configure_services</td>
<td>Ensure systemd services and timers specified in borgmatic_services are configured</td>
</tr>
<tr>
<td>borgmatic_user</td>
<td>User that runs borgmatic</td>
</tr>
<tr>
<td>borgmatic_local_ssh_options</td>
<td>SSH command line options used by control node when managing remote backup servers</td>
</tr>
<tr>
<td>borgmatic_remote_public_keys</td>
<td>Public keys belonging to backup servers to be registered in the known_hosts file</td>
</tr>
<tr>
<td>borgmatic_configs</td>
<td>Borgmatic config details</td>
</tr>
<tr>
<td>borgmatic_services</td>
<td>Systemd service and timer details</td>
</tr>
<tr>
<td>borgmatic_remote_options</td>
<td>Host specific options enforced by remote backup servers</td>
</tr>
<tr>
<td>borgmatic_extract</td>
<td>Extract details</td>
</tr>
<tr>
<td>borgmatic_perform_extract</td>
<td>Perform extract of backup data as defined in borgmatic_extract</td>
</tr>
</table>

`borgmatic_configs`, `borgmatic_services`, `borgmatic_remote_options` and `borgmatic_extract` can include multiple items following the same structure.

It is recommended that sensitive variables be stored in a [vault]. Variables shared between hosts can be placed in a group vault such as `group_vars/all.yml`, and host specific variables can be placed in a host specific vault such as `host_vars/inventory_hostname.yml`.

### Extracting Backup Data

To extract data from a previous backup, set `borgmatic_extract`:

```yaml
borgmatic_extract:
  # Repository to extract from
  - repository: user@example.com:/path/to/repo
  
    # Optional, specify the archive,
    # defaults to 'latest'
    archive: latest
  
    # Optional, specify a list of source paths
    # from within the archive to extract
    # defaults to entire archive
    paths:
      - /home/username
      - /etc
  
    # Destination that will be extracted into
    destination: '/'
```

To perform the extract, set `borgmatic_perform_extract=true` at playbook runtime:

```sh
ansible-playbook playbook.yml -e 'borgmatic_perform_extract=true'
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
# Variables for specific host

---

borgmatic_configs:
  # Typical setup for backups to a remote server
  - path: /etc/borgmatic.d/remote.yaml
    state: present
    encryption: repokey-blake2
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

      hooks:
        healthchecks: https://hc-ping.com/your-uuid-here

  # Typical setup for backups to a removable drive
  # Repository initialization will fail if the removable drive is not mounted
  - path: /etc/borgmatic.d/local.yaml
    state: present
    encryption: authenticated-blake2
    content:
      location:
        source_directories:
          - /home/username

        repositories:
          - /run/media/username/backup

      storage:
        encryption_passphrase: redacted

      retention:
        keep_hourly: 24
        keep_daily: 7
        keep_weekly: 4
        keep_monthly: 6
        keep_yearly: 1

      hooks:
        # Exit if external drive is not mounted
        before_backup:
          - findmnt /run/media/username/backup > /dev/null || exit 75

borgmatic_services:
  - name: borgmatic-remote-backup
    state: present
    config: /etc/borgmatic.d/remote.yaml
    actions:
      - prune
      - compact
      - create
    timer:
      oncalendar: daily
 
  # Integrity checks are time consuming, and can be run less frequently
  - name: borgmatic-remote-check
    state: present
    config: /etc/borgmatic.d/remote.yaml
    actions:
      - check
    timer:
      oncalendar: weekly
      randomizeddelaysec: 2h

borgmatic_remote_options:
  - remote: user@example.com
    remote_path: borg1
    append_only: true
    restrict:
      repositories:
        - /path/to/repo

borgmatic_extract:
  - repository: user@example.com:/path/to/repo
    destination: '/'
```

[borg]: https://www.borgbackup.org/
[borgmatic]: https://torsion.org/borgmatic/
[rsync.net]: https://www.rsync.net/products/borg.html
[repository-restricted]: https://borgbackup.readthedocs.io/en/stable/usage/serve.html
[rsync.net web interface]: https://www.rsync.net/am/dashboard.html
[vault]: https://docs.ansible.com/ansible/latest/cli/ansible-vault.html
