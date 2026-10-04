# Docker Compose Application and Data Migration Procedure

Use this runbook to migrate a Docker Compose application and its persistent data from a source server to a destination server.

The normal migration path is:

1. Preserve the original Compose project and image references.
2. Stop writes and create consistent backups.
3. Transfer the migration bundle once and verify its checksums.
4. Let Docker Compose create the destination volumes.
5. Confirm every destination volume is empty.
6. Restore the data and start the Compose project.

> **Critical safety rules**
>
> - Never run `docker compose down -v` during the migration. The `-v` option removes named volumes declared by the project and can permanently delete application data.
> - Do not run `docker volume prune` or remove a source volume during the migration.
> - Do not manually create destination volumes for the normal Compose migration path. Let `docker compose create` create them.
> - Do not restore into a non-empty destination volume. Extraction can overwrite files with matching paths and leave unrelated files in place, producing a mixed dataset.
> - Keep the source application, source volumes, and verified backup bundle intact until the destination has passed acceptance testing.
> - Prevent the source and destination from accepting writes at the same time unless the application explicitly supports multi-node operation.

## Scope and assumptions

This procedure assumes:

- the application is, or can be, represented by a Compose project;
- Docker Engine and Docker Compose are installed on both servers;
- you have SSH access and enough free space for the backup archives;
- named volumes use a driver that can be backed up through a temporary container; and
- application configuration, secrets, certificates, and bind-mounted files can be transferred securely.

Replace every value in angle brackets, such as `<service-name>`, before running a command. Run Compose commands from the project directory and use the same Compose files, profiles, environment file, and project name throughout the migration.

## Backup and restore are opposite operations

The volume commands look similar, but the `tar` operation and read-only mount are different:

| Operation | Server | Data direction | `tar` option | Read-only mount |
|---|---|---|---|---|
| Backup | Source | volume to archive | `-czf` — **create** gzip archive | source volume (`:ro`) |
| Restore | Destination | archive to volume | `-xzf` — **extract** gzip archive | archive directory (`:ro`) |

```text
BACKUP:  source volume  -- tar -czf -->  archive
RESTORE: archive        -- tar -xzf -->  destination volume
```

## Compose volume ownership and naming

These declarations have different lifecycle behavior:

### Compose-managed, project-scoped name

```yaml
services:
  app:
    volumes:
      - app-data:/var/lib/app

volumes:
  app-data:
```

Compose normally creates an engine volume such as `<project>_app-data`. It adds project metadata and manages the volume as part of the Compose application.

### Compose-managed, explicit engine name

```yaml
services:
  app:
    volumes:
      - app-data:/var/lib/app

volumes:
  app-data:
    name: app-data
```

Compose creates and manages the volume if it does not exist, but the engine-level name is exactly `app-data` rather than `<project>_app-data`. Explicit names are useful when stable names are required, but they are not scoped by project name and can collide with another project.

### Externally managed volume

```yaml
services:
  app:
    volumes:
      - app-data:/var/lib/app

volumes:
  app-data:
    name: app-data
    external: true
```

Compose expects this volume to exist already and does not manage its lifecycle. Use `external: true` only when the volume is intentionally managed outside this Compose project. It is not the normal choice for this migration.

## Phase 1 — Prepare and inventory the source

### 1. Locate the authoritative Compose project

Identify and protect the complete deployment definition:

- `compose.yaml`, `compose.yml`, or `docker-compose.yml`;
- any override or included Compose files;
- `.env` and other environment files;
- application configuration;
- certificates and secrets, transferred only through an approved secure method;
- Dockerfiles and build contexts for locally built images; and
- scripts or systemd units used to launch the project.

If the project uses an explicit project name, record it. Resource names can change if the destination uses a different directory or project name.

```bash
docker compose ls
docker compose config --volumes
docker compose config --images
```

Use the same `-p <project-name>`, `COMPOSE_PROJECT_NAME`, `--env-file`, `-f`, and `--profile` options on all subsequent Compose commands when the deployment requires them.

### 2. Record the effective configuration and runtime details

Create a protected migration working directory. The expanded Compose configuration and inspection output can contain credentials, tokens, and other sensitive values.

```bash
MIGRATION_DIR="$(pwd)/docker-migration"
mkdir -p "${MIGRATION_DIR}/records" "${MIGRATION_DIR}/volumes"

docker compose config > "${MIGRATION_DIR}/records/compose-expanded.yaml"
docker compose ps -a > "${MIGRATION_DIR}/records/compose-ps.txt"
docker compose config --images > "${MIGRATION_DIR}/records/images.txt"
docker compose config --volumes > "${MIGRATION_DIR}/records/volumes.txt"
```

Save the original Compose files and required configuration in the migration directory. Do not omit files referenced by `env_file`, `secrets`, `configs`, `include`, bind mounts, or build contexts.

For each container, record the exact image reference and the effective mounts, networks, configuration, and host settings:

```bash
docker inspect <container-name> --format '{{.Config.Image}}'
docker inspect <container-name> --format '{{json .Mounts}}' | jq
docker inspect <container-name> --format '{{json .NetworkSettings.Networks}}' | jq
docker inspect <container-name> --format '{{json .Config}}' | jq
docker inspect <container-name> --format '{{json .HostConfig}}' | jq
```

Also record any deployment details that are not represented in Compose, including firewall rules, reverse-proxy configuration, DNS, scheduled jobs, device mappings, external networks, and storage-driver requirements.

### 3. Identify every persistent-data location

Review each mount in the inspection output and classify it as:

- a named volume;
- a bind-mounted host path;
- an anonymous volume; or
- temporary container-writable storage.

Do not assume `docker commit` captures mounted data. Data stored in named volumes, anonymous volumes, or bind mounts is outside the container image and must be migrated separately.

Anonymous volumes should normally be converted to named Compose volumes before relying on this runbook. If that is not possible, record their engine names and treat them as named-volume backups for this migration.

### 4. Make databases and transactional applications consistent

A filesystem copy of a live database volume may be inconsistent even if the archive command succeeds.

For databases and other transactional applications:

1. Follow the vendor's supported backup procedure.
2. Create a native logical or physical backup when appropriate, such as `pg_dump`, `pg_dumpall`, `mysqldump`, `mysqlpump`, or `mongodump`.
3. Store that native backup in the migration bundle as an additional recovery path.
4. Stop application writes before taking the raw volume backup unless the product explicitly supports filesystem-level hot backups or a storage snapshot procedure is being used.

Quiesce the application and stop the Compose services:

```bash
docker compose stop
```

> **Prohibited:** Do not add `-v`. Do not run `docker compose down -v`.

Confirm that no container or external process is still writing to the volumes before continuing.

## Phase 2 — Back up the source

### 5. Back up each named volume

For every source volume, set the actual engine volume name and a filesystem-safe archive name. Mount the source volume read-only.

```bash
SOURCE_VOLUME="<actual-source-volume-name>"
ARCHIVE_NAME="<archive-name>.tar.gz"

docker run --rm \
  -v "${SOURCE_VOLUME}:/data:ro" \
  -v "${MIGRATION_DIR}/volumes:/backup" \
  alpine:3.20 \
  tar -czf "/backup/${ARCHIVE_NAME}" -C /data .
```

This is a **backup** because `tar -czf` creates the archive. The temporary container can write only to the backup directory; the application volume is read-only.

Repeat the command for every named or anonymous volume required by the application. Record the mapping from source engine volume name to archive name and destination logical volume in `records/volume-map.txt`.

### 6. Back up bind-mounted data

For each bind-mounted directory, archive the contents from the host after writes have stopped:

```bash
sudo tar -czf "${MIGRATION_DIR}/<bind-archive>.tar.gz" \
  -C <source-bind-path> .
```

Record the source path, intended destination path, owner, group, mode, ACLs, extended attributes, and security labels where applicable. A basic `tar` archive may not preserve every platform-specific ACL, extended attribute, or SELinux context; use appropriate `tar` options or `rsync` when those are required.

### 7. Use original images and configuration

For a normal migration, use the image references and build definitions in the Compose project:

- pull registry images on the destination with `docker compose pull`; and
- rebuild locally built images from their Dockerfiles and build contexts with `docker compose build` when reproducible builds are available.

Prefer immutable image digests or otherwise confirm that a mutable tag still points to the intended image.

Do not use `docker commit` as the normal migration method. A committed container image can hide undocumented drift, does not capture volume or bind-mounted data, and is harder to reproduce or maintain.

If the destination cannot access the registry, save the original images for offline transfer:

```bash
docker image save $(docker compose config --images) | gzip \
  > "${MIGRATION_DIR}/images.tar.gz"
```

The legacy `docker commit` procedure is described separately near the end of this document and should be used only when the running container contains necessary, undocumented changes that cannot be reconstructed.

### 8. Generate checksums

After every required file has been placed in the migration directory, generate a checksum manifest:

```bash
cd "${MIGRATION_DIR}"
find . -type f ! -name SHA256SUMS -print0 \
  | sort -z \
  | xargs -0 sha256sum > SHA256SUMS
```

Review the bundle before transfer. It should contain the Compose project, required configuration, volume and bind-mount archives, native database backups, inventory records, and `images.tar.gz` only when offline image transfer is necessary.

### 9. Transfer the migration bundle once

Transfer the complete directory with one command. Do not separately copy `*.tar.gz` and then copy the same image archive again.

```bash
cd "$(dirname "${MIGRATION_DIR}")"
scp -r "$(basename "${MIGRATION_DIR}")" \
  <user>@<destination-host>:<destination-parent-directory>/
```

For large or restartable transfers, `rsync` is usually more suitable:

```bash
rsync -a --info=progress2 "${MIGRATION_DIR}/" \
  <user>@<destination-host>:<destination-migration-directory>/
```

Use one transfer method, not both.

## Phase 3 — Prepare the destination with Compose

### 10. Verify the transferred bundle

On the destination, verify the complete bundle before loading images or restoring data:

```bash
cd <destination-migration-directory>
sha256sum -c SHA256SUMS
```

Every entry must report `OK`. If any checksum fails, stop and repeat the transfer of the affected bundle before continuing.

### 11. Install and validate Docker

Confirm that the destination has compatible Docker Engine and Compose versions, sufficient disk space, the required volume drivers, and access to any external networks or storage services.

```bash
docker version
docker compose version
docker info
```

### 12. Place and validate the Compose project

Copy the Compose project to its intended permanent location. Restore environment files, configuration, certificates, and secrets through their approved secure process. Use the original project name or explicitly set it so automatically generated resource names remain predictable.

Validate the effective destination configuration before creating anything:

```bash
cd <destination-compose-project-directory>
docker compose config --quiet
docker compose config --images
docker compose config --volumes
```

Review the top-level `volumes` declarations. For Compose-managed volumes, do not add `external: true` and do not run `docker volume create`.

### 13. Pull, build, or load the original images

When registry access is available:

```bash
docker compose pull
```

Build locally defined images when required:

```bash
docker compose build
```

For an offline migration bundle:

```bash
gunzip -c <destination-migration-directory>/images.tar.gz | docker image load
```

Confirm that the resulting image references match those required by the Compose configuration.

### 14. Let Compose create its resources without starting services

From the destination project directory, create the containers, networks, and Compose-managed volumes:

```bash
docker compose create
```

Do not manually create the volumes first. Manual pre-creation can make resource ownership ambiguous and may lead to an incorrect `external: true` workaround.

List the created volumes and inspect each created service container to map the logical Compose volume to its actual engine name and container destination:

```bash
docker volume ls
docker inspect "$(docker compose ps -a -q <service-name>)" \
  --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
```

For a default declaration such as `app-data:`, expect an engine name similar to `<project>_app-data`. For `name: app-data`, expect exactly `app-data`.

## Phase 4 — Verify empty volumes and restore data

### 15. Verify every destination volume is empty

Some images can copy files from an image directory into a newly attached empty volume. A previous migration attempt may also have left data behind. Therefore, verify the actual contents after `docker compose create` and before extraction.

```bash
DEST_VOLUME="<actual-destination-volume-name>"

docker run --rm \
  -v "${DEST_VOLUME}:/data:ro" \
  alpine:3.20 \
  sh -c '
    if [ -n "$(find /data -mindepth 1 -print -quit)" ]; then
      echo "NOT EMPTY: refusing restore"
      find /data -mindepth 1 -maxdepth 2 -print
      exit 1
    fi
    echo "EMPTY: safe to restore"
  '
```

The command must print `EMPTY: safe to restore` and exit successfully.

If it reports `NOT EMPTY`, stop. Determine whether the files were seeded by the image, belong to an earlier migration attempt, or are real application data. Do not extract the archive over them. Resolve the volume state deliberately and repeat the emptiness check before restoring.

### 16. Restore each named volume

Mount the destination volume read-write and the directory containing the archive read-only:

```bash
DEST_VOLUME="<actual-destination-volume-name>"
ARCHIVE_NAME="<archive-name>.tar.gz"
BACKUP_DIR="<destination-migration-directory>/volumes"

docker run --rm \
  -v "${DEST_VOLUME}:/data" \
  -v "${BACKUP_DIR}:/backup:ro" \
  alpine:3.20 \
  tar -xzf "/backup/${ARCHIVE_NAME}" -C /data
```

This is a **restore** because `tar -xzf` extracts the archive. The archive directory is read-only so the restore container cannot modify the backup.

> `tar -xzf` is not overwrite-protected. If a path already exists in the destination, extraction can overwrite it. Files that exist only in the destination are generally not removed, so restoring into a non-empty volume can combine old and restored data. The mandatory emptiness check prevents this unsafe mixed state.

Repeat the emptiness check and restore for every named volume. Do not start any application service until all related volumes have been restored.

### 17. Restore bind-mounted data

Create each destination host directory with the expected parent-path permissions, verify it is empty, and extract its archive:

```bash
sudo mkdir -p <destination-bind-path>
sudo find <destination-bind-path> -mindepth 1 -print -quit
sudo tar -xzf <destination-migration-directory>/<bind-archive>.tar.gz \
  -C <destination-bind-path>
```

The `find` command must produce no output before extraction. Verify ownership, permissions, ACLs, extended attributes, and SELinux/AppArmor requirements against the source records.

### 18. Verify restored data before startup

Compare the restored layout and ownership with the source inventory. At minimum, inspect the top-level files and confirm that the expected application UID/GID can read and write the restored paths.

```bash
docker run --rm \
  -v "${DEST_VOLUME}:/data:ro" \
  alpine:3.20 \
  sh -c 'ls -lan /data; find /data -mindepth 1 -maxdepth 2 -print | head -100'
```

For critical data, perform an application-specific integrity check or a trial restore of the native database backup before cutover.

## Phase 5 — Start and validate the destination

### 19. Start the Compose project

```bash
cd <destination-compose-project-directory>
docker compose up -d
```

Compose is the preferred startup method because it recreates the declared services, networks, health checks, capabilities, devices, labels, logging options, secrets, and other configuration consistently. Reconstructing a non-trivial deployment with an ad hoc `docker run` command is error-prone.

### 20. Validate the migration

```bash
docker compose ps
docker compose logs --tail=200
```

Complete application-specific checks:

- all services are running or healthy;
- expected ports and reverse-proxy routes work;
- application login and primary workflows succeed;
- migrated records, uploads, and configuration are present;
- the application can write new test data;
- background jobs and integrations work;
- database integrity checks pass; and
- monitoring, backups, and scheduled jobs are enabled on the destination.

Compare the destination mounts, networks, image references, and runtime settings with the saved source inspection records.

### 21. Cut over and retain rollback capability

Move traffic to the destination only after validation. Keep the source stopped but intact during the agreed rollback window.

If destination validation fails:

1. stop destination services with `docker compose stop`;
2. prevent clients from reaching the destination;
3. investigate or rebuild the destination from the verified backup bundle; and
4. restart the source only after confirming that doing so will not create a split-brain or conflicting-write condition.

Do not delete source containers, source volumes, or migration archives until the migration owner has accepted the destination and the rollback window has expired.

## Alternative — Direct streamed volume copy

This can be useful when both servers are reachable and there is insufficient local space for an archive. It is less resilient than a verified archive because interruption requires restarting the copy and there is no retained recovery artifact.

First, run `docker compose create` on the destination and pass the mandatory emptiness check. Stop source writes. Then stream from the read-only source volume into the empty Compose-created destination volume:

```bash
docker run --rm \
  -v <source-volume>:/from:ro \
  alpine:3.20 \
  tar -czf - -C /from . \
| ssh <user>@<destination-host> \
  'docker run --rm -i -v <destination-volume>:/to alpine:3.20 tar -xzf - -C /to'
```

Validate the destination contents before starting Compose. Prefer the archive-and-checksum method when practical.

## Legacy or offline exception — Commit and save

Use this only when a running legacy container contains necessary changes that are not represented in a Dockerfile, Compose configuration, or registry image.

Before committing:

- record the original image, configuration, mounts, networks, and host settings;
- stop application writes;
- understand that mounted volume and bind-mount data is **not** included; and
- back up all persistent data separately using the earlier steps.

```bash
docker commit -p <container-name> <legacy-image-name>:migration
docker image save <legacy-image-name>:migration | gzip \
  > "${MIGRATION_DIR}/legacy-image.tar.gz"
```

Regenerate `SHA256SUMS` after adding the image archive. On the destination, verify checksums and load it:

```bash
gunzip -c <destination-migration-directory>/legacy-image.tar.gz \
  | docker image load
```

Update the destination Compose file to reference the loaded image, and document the exception. After recovery, create a Dockerfile or other reproducible build definition so future migrations do not depend on `docker commit`.

## Troubleshooting

| Symptom | Likely cause | Safe response |
|---|---|---|
| Compose warns that a volume already exists but was not created by the project | The volume was manually created or belongs to another project | Stop. Confirm its origin. For a new migration, use a clean Compose-created volume; do not add `external: true` merely to silence the warning. |
| Emptiness check reports files | Image copy-up, an earlier attempt, or existing application data | Do not restore. Identify the files and resolve the volume state before retrying. |
| Checksum fails | Incomplete or corrupt transfer | Stop and transfer the bundle again. Do not restore from the failed file. |
| Permission errors after restore | UID/GID, modes, ACLs, or security labels differ | Compare numeric ownership with the source and apply the application/vendor-supported correction. |
| Application starts without its data | Wrong engine volume or wrong container mount path | Compare the destination container mounts with `records/volume-map.txt` and source inspection output. |
| Database recovery or consistency errors | Live filesystem copy or incompatible database/image version | Keep the source intact; follow the database vendor's recovery process or restore the native database backup. |
| Destination uses unexpected volume/network names | Different Compose project name, directory, files, or profiles | Re-run with the original project name and exact Compose options; verify before restoring. |

## Completion checklist

- [ ] Original Compose files, environment files, configuration, and required secrets are available.
- [ ] Original image references, mounts, networks, configuration, and host settings are recorded.
- [ ] Every persistent-data location is classified and included.
- [ ] Database-native backups were created where appropriate.
- [ ] Source writes were stopped before raw volume backup.
- [ ] Source volumes were mounted read-only during backup.
- [ ] `tar -czf` was used for backup and `tar -xzf` only for restore.
- [ ] Images came from the registry or reproducible builds, or an offline image archive was justified.
- [ ] A checksum manifest was generated and successfully verified on the destination.
- [ ] The migration bundle was transferred once without duplicate `scp` commands.
- [ ] Destination volumes were created by `docker compose create`, not manually.
- [ ] Every destination volume was proven empty before extraction.
- [ ] The archive directory was mounted read-only during restore.
- [ ] Bind mounts, ownership, permissions, ACLs, and security labels were restored as required.
- [ ] The destination passed application and data-integrity checks.
- [ ] The source and verified backup bundle remain available through the rollback window.
- [ ] `docker compose down -v` was not used at any point.

## References

- [Docker Compose volume declarations](https://docs.docker.com/reference/compose-file/volumes/)
- [Docker volume storage](https://docs.docker.com/engine/storage/volumes/)
- [`docker compose down` and the destructive `--volumes` option](https://docs.docker.com/reference/cli/docker/compose/down/)

