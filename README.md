# kube-loop-flexvolume

This driver allows you to create large files on shared filesystem, then format it and mount as volume for container.

It can be a necessary step if your containers have a lot of small files and you want to avoid excessive access to metadata.

## Features

* Automatic volumes allocation and creation
* Multi-mount protection (for ext4 only)
* Checking for shared filesystem (is mounted or no)

## Requirements

* Kubernetes: >=1.9 version
* Common filesystem mounted on each node in one place

## Limitations

* Each volume will formatted with simple file system, so only one pod can use it in r/w mode.

## Quick start

For installation just run:

```
kubectl create -f https://raw.githubusercontent.com/kvaps/kube-loop-flexvolume/master/kube-loop-flexvolume.yaml
```

## Usage

### Simple example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: test
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: test
    flexVolume:
      driver: "kvaps/loop"
      fsType: "ext4"
      options:
        file: "/tmp/kube-volumes/test"
        size: "30m"
```

This example will create new volume in `/tmp/kube-volumes/test`.

### Shared filesystem example:

In case if you have shared filesystem mounted on `/stor/myshare`, and you want to create `/stor/myshare/kube-volumes/test`, please use following example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: test
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: test
    flexVolume:
      driver: "kvaps/loop"
      fsType: "ext4"
      options:
        share: "/stor/myshare"
        file: "/kube-volumes/test"
        size: "1g"
```

It will protect you new volume creation if your share will unmounted for some reason.

### Checking

Run these commands from node which runs your container, for make sure that it is working fine:

```bash
# fdisk -l /stor/myshare/kube-volumes/test
Disk /stor/myshare/kube-volumes/test: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
```bash
# wipefs /stor/myshare/kube-volumes/test
offset               type
----------------------------------------------------------------
0x438                ext4   [filesystem]
                     UUID:  54d37b47-b1af-49be-a3a8-a6cebb60a443
```
```bash
# cat /proc/mounts | grep loop
/dev/loop0 /data/local/data/kubelet/pods/45ed4ce5-fb12-11e7-9d64-024277fad76c/volumes/kvaps~loop/test ext4 rw,relatime,data=ordered 0 0
/dev/loop0 /var/lib/kubelet/pods/45ed4ce5-fb12-11e7-9d64-024277fad76c/volumes/kvaps~loop/test ext4 rw,relatime,data=ordered 0 0
```

If something goes wrong, you can use `kubect describe pod` feature and check `kubelet` log.

## License

* Kube-loop-flexvolume is under the Apache 2.0 license. (See the [LICENSE](LICENSE) file for details)
