# Podman - Runtime

## Issue 102
### Insufficient storage for the Remote Engine container image

```shell
Error: writing blob: adding layer with blob "sha256:a8698704d0df"
docker run return code: 125.
```

#### Resolution:
Docker/Podman images are stored in `/var` by default. It is recommended to have at least **50 GB** of free space in `/var` for installing the Remote Engine container image.

Check the available storage:

```shell
df -h /var
```

Example output:
```shell
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda4       8.8G 42.4G 52.8G 81% /
```

#### Option 1:
To clean up disk space, run:

```shell
podman system prune
```

> [!IMPORTANT]
> This removes all unused images, including manually pulled ones.

> [!TIP]
> Caution your client on the purpose of this command before executing.

---

## Issue 103
### Insufficient subgids/subuids for user namespace

```shell
Error: writing blob: adding layer with blob "sha256:a9089747d5ad"
```

#### Resolution:
Rootless Podman requires a specific range of UIDs (e.g., **1000321001**). Follow these steps:

1. Install `shadow-utils`:

```shell
sudo yum -y install shadow-utils
```

2. Edit `/etc/subuid`:

```shell
sudo vi /etc/subuid
```

Add:
```text
eltpoc:100000:1001321001
```

3. Verify the change:

```shell
cat /etc/subuid
```

4. Edit `/etc/subgid`:

```shell
sudo vi /etc/subgid
```

Add:
```text
eltpoc:100000:1001321001
```

5. Verify:

```shell
cat /etc/subgid
```

6. Apply changes:

```shell
podman system migrate
```

> [!TIP]
> Ensure the UID/GID ranges are properly configured and match across both files.

---

## Issue 104
### Insufficient 'cpu' and 'cpuset' permissions

```shell
Error: OCI runtime error: crun: the requested cgroup controller
```

#### Resolution:
Non-root users lack resource limit delegation permissions. Fix as follows:

1. Verify delegation:

```shell
cat "/sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u)"
```

Expected output includes:
```text
memory
pids
```

2. If `cpu` and `cpuset` are missing, edit the `.conf` file:

```shell
sudo vi /etc/systemd/system/user@.service.d/delegate.conf
```

Add:
```ini
[Service]
Delegate=memory pids cpu cpuset
```

3. Logout and back in to test.

> [!TIP]
> After re-logging, re-run the verification command to confirm `cpu` and `cpuset` are now listed.

---

## Issue 105
### Remote Engine not starting after 250 seconds (init.sh: Permission Denied)

```shell
waiting for amin-aws_runtime to start... time elapsed: 245 seconds
waiting for amin-aws_runtime to start... time elapsed: 250 seconds
ERROR: Could not start container amin-aws_runtime in 250 seconds
```

#### Resolution:
Missing executable permissions on the volume directory can cause this.

> [!NOTE]
> If the `--volume-dir` flag is not specified, `dsengine.sh` uses `/tmp/docker/volumes` as the default volume directory.

1. Delete existing containers:

```shell
podman ps -a
podman rm -f "IMAGE_NAME"
```

2. Verify volume directory permissions:

```shell
whoami
ls -ld /tmp/docker/volumes
```

If permissions are incorrect:

```shell
sudo chmod -R 755 /tmp/docker/volumes
```

3. Alternatively, specify a custom volume directory:

```shell
export VOLUME_DIR=test/docker/volumes
./dsengine.sh start -n "$ENGINE_NAME" \
  -a "$CLOUD_API_KEY" \
  -e "$ENCRYPTION_KEY"
```

4. Disable SELinux enforcement if needed:

```shell
./dsengine.sh start -n "$ENGINE_NAME" \
  --security-opt label=disable
```

> [!IMPORTANT]
> SELinux can block container access even with correct mounts. Disabling SELinux (`--security-opt`) can bypass this restriction.

---

## Issue 201
### Logs not writing to IBM Cloud

#### Resolution:
> [!NOTE]
> _Credit to Pat Marcello!_

---

## Issue 202
### Job failure (Local hostname binding in /etc/hosts)

```text
**** Startup error on node1 connecting with conductor on 172.28
##E IIS-DSEE-TFPM-00148
**** Startup error on node2 connecting with conductor on 172.28
```

#### Resolution:
Job execution fails due to incorrect host IP mappings. Fix it:

1. Find the container:

```shell
docker ps
# OR
podman ps
```

2. Exec into the container:

```shell
sudo podman exec -it <container-name-or-id> bash
```

3. Edit `/etc/hosts`:

```shell
nano /etc/hosts
```

4. Remove the host IP mapping line.

> [!IMPORTANT]
> Ensure only the incorrect mappings are removed to avoid breaking network connections.

---

**End of Document**
