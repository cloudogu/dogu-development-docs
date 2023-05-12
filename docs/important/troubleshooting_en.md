# Troubleshooting

The Troubleshooting chapter provides assistance in locating possible errors during Dogu development.
Solutions are also presented for known errors.

## How do I avoid errors due to invalid configuration entries?

- Valid registry paths are to be considered
- `doguctl` can validate configuration values. See [Validation](relevant_functionalities_en.md#validation-and-default-values)
- Overview of all keys: `etcdctl ls -r config/<doguname>`
- From the container it is possible to check encrypted values:
    - `docker exec -it redmine doguctl config -e sa-postgresql/username`

## The dogu does not start and/or is in a restart loop.

If there is an approximate guess about the position of the faulty location, additional `echo` output can help to analyze the problem.

If the source of the error is still unclear, it is necessary to execute the statements inside the container one by one:
- Shell into the container: `docker exec -it <doguname> bash`.
- After that, the container can be debugged:
    - Checking the filesystem
    - Iterative execution of the commands in the scripts to locate the defect
    - Repairing the defect using on-board means in the container

During a restart loop, you have the following options to bring the container to a stable state:
- During development, the best way you can do is to overwrite the entrypoint of the container.
    - `ENTRYPOINT ["tail", "-f", "/dev/null"]` instead of `CMD ["/resources/startup.sh"]`.
    - Alternatively, a `sleep --infinite` can be written anywhere in the startup script.
- In production, a container must be stopped manually in a restart loop and restarted with a commit:
```bash
docker stop ldap
# commit the stopped container to a local debug image
docker commit hashFromYourStoppedContainer_cebf8699cd9d debug/ldap
# the volume arguments can be generated with docker inspect
docker inspect ldap -f '{{range.Mounts}}-v {{.Source}}:{{.Destination}} {{end}}'
# debug your dogu container
docker run -ti \
--rm \
# here goes the docker volume list from above
-v /etc/ces:/etc/ces \
-v /var/lib/ces/ldap/volumes/_private:/private \
-v /var/lib/ces/ldap/volumes/config:/etc/openldap/slapd.d \
-v /var/lib/ces/ldap/volumes/db:/var/lib/openldap \
--entrypoint=/bin/sh --network cesnet1 --name ldap --hostname ldap \
debug/ldap
```

## Logging output is missing or has wrong granularity

There can be many reasons why there is completely/partially missing logging output or the wrong logging output occurs.

- Is an incorrect path used to read the log level?
  - The correct path is `config/<doguname>/logging/root`.
- Is an invalid log level being used?
  - The four valid values are: `ERROR`, `WARN`, `INFO`, `DEBUG`.
- Is the value not interpreted in a meaningful way when starting the Dogu?
   - A correct example would be:
```bash
function mapDoguLogLevel() {
  local DEFAULT_LOGGING_KEY="logging/root"
  local LOG_LEVEL_ERROR="ERROR"
  local LOG_LEVEL_WARN="WARN"
  local LOG_LEVEL_INFO="INFO"
  local LOG_LEVEL_DEBUG="DEBUG"
  local DEFAULT_LOG_LEVEL=${LOG_LEVEL_WARN}
  
  echo "Mapping dogu specific log level"
  local currentLogLevel=$(doguctl config --default "${DEFAULT_LOG_LEVEL}" "${DEFAULT_LOGGING_KEY}")

  echo "${SCRIPT_LOG_PREFIX} Mapping ${currentLogLevel} to Catalina"
  case "${currentLogLevel}" in
    "${LOG_LEVEL_ERROR}")
      export CATALINA_LOGLEVEL="SEVERE"
    ;;
    "${LOG_LEVEL_INFO}")
      export CATALINA_LOGLEVEL="INFO" ;; "${log_level_error}")
    ;;
    "${LOG_LEVEL_DEBUG}")
      export CATALINA_LOGLEVEL="FINE" ;; "${log_level_debug}")
    ;;
    *)
      echo "${SCRIPT_LOG_PREFIX} Falling back to WARNING"
      export CATALINA_LOGLEVEL="WARNING" ;;.
    ;;
  esac
}
```

## Volumes

It is advisable to check the size of the volumes regularly.
These are mounted under the path `/var/lib/ces/<dogu>/volumes/`.

A Dogu should store a growing dataset exclusively in Docker volumes described in the [`dogu.json`](https://github.com/cloudogu/dogu-development-docs/blob/main/docs/core/compendium_en.md#volumes).

Volumes have other advantages besides better speed: they are not lost when the container is terminated. It is also easier to monitor disk usage.

## Where can I find FQDN, certificate and other general configurations?

see: [registry entries](relevant_functionalities_en.md#other-registry-entries)

## How are registry entries read/written?

By means of [doguctl](relevant_functionalities_en.md#usage-of-doguctl).
It is included in our [base-image](https://github.com/cloudogu/base).

## How can a container be re-instantiated?

- `cesapp recreate <doguname>`

## The dogu is not accessible via Nginx.

- Was this environment variable set in the Dockerfile?
    - `ENV SERVICE_TAGS=webapp`
- Check configuration in the Nginx container
    - `docker exec -it nginx bash`
    - `cat /etc/nginx/conf.d/app.conf`

## Dogu does not appear in the warp menu

- For an entry in the warp menu, the tag `warp` and a category must be defined in dogu.json.
- The dogu must deliver valid HTML with a trailing `<\html>` tag.
- The dogu was not registered due to an incorrect installation.

## Dogu volumes are not respected by Backup & Restore

In order for volume data of a dogu to be respected by the Backup & Restore mechanism, the corresponding volume must be extended with the `NeedsBackup` field:

```json
{
  "Name": "db",
  "Path":"/var/lib/openldap",
  "NeedsBackup": true
}
```

## Freeing disk space

- Removal of logs no longer needed e.g. syslog
  - `> /var/log/syslog`
- Rotate logs
  - `logrotate --force /etc/logrotate.conf`
- Removing Docker images that are no longer needed
  - `docker rmi my-image:1.2.3`

## The filesystem is full although `df` does not reflect this.

In this case it may be helpful to balance the Btrfs filesystem.

- `btrfs filesystem balance /var/lib/docker/btrfs`

**caution**:
- The process puts the partition in a read-only mode, which is not reset on abort.
In that case a system reboot is necessary.

## In which directories is the most space consumed?

- `cd <dir> && du -d 1 -h .* | sort -h`

# Resynchronize Ubuntu time

If the Cloudogu EcoSystem is offline for long periods of time, it may be necessary to resynchronize the system time:

- `systemctl restart systemd-timesyncd.service`