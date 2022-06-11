# collect

Docker container to `rsync` remote directories from multiple machines on a network into one directory. The **push** container will then `rclone` from that consolidated directory out to your favorite cloud provider e.g. backblaze.

## installation

### assemble your `fstab`

The **collect** container will mount remote sources which it will later `rsync` from. In order to mount them, you'll need to supply a standard Linux `fstab` file. An example is provided here as `fstab.example`. You can use anything that `fstab` supports including `sshfs` for Linux or Mac, or `cifs` to mount Windows machines.

When you're done, a file called `fstab` must exist in the same directory as the `Dockerfile`.

### assemble your `sources`

`sources` is a file used by the **collect** container to describe what should be `rsync`'d to where.

It can consist of any number of rows. Each row must consist of an `ip`, a `mountpoint` and an `rsync_destination`.

The **collect** container will ping each `ip` every ten minutes. If an `ip` is up, it will start (resume) an `rsync` from the `mountpoint` to the `rsync_destination`. The `mountpoint` in `sources` must match a `mountpoint` in `fstab`.

All of the `rsync_destination`s should be a subdirectory of the container's `/backup` directory. Typically, each `rsync_destination` in **collect**'s `sources` will match a `source` in the **push** container's `buckets` file. Thus the **collect** container `rsync`s multiple remote directories into a single place, and the **push** container `rclone`s that single place out to the cloud.

When you're done, a file called `sources` must exist in the same directory as the `Dockerfile`.

### How can this work? These containers are independent!

It works because the `/backup` directory in both the **collect** and **push** containers are mounted as volumes to the same single location on the machine running the two containers. Note that therefore that the **collect** and **push** containers must run on a single machine so the directory can be shared.

Let's say you run both containers on a Synology NAS. The `/backup` volume in **collect** and the `/backup` volume in **push** are mounted to the same single directory on the NAS. Therefore **collect** accumulates files in this directory; **push** grabs stuff under that directory and pushes out to the cloud.

### (`sshfs` only) assemble your `known_hosts`

If you're not using `sshfs` in your `fstab` you can skip this step.

You can also skip this step if you don't care too much about security(!!), by adding `,StrictHostKeyChecking=no,` to `sshfs` entries in your `fstab`. I wouldn't recommend it.

For the **collect** container to be able to use `sshfs` to mount remote hosts it needs to "know" all of the remote hosts that it will be mounting.

To do that it needs a consolidated list of all of the remote hosts' IPs and public keys in a single file called `known_hosts`.

Each line of `known_hosts` consists of two parts: the IP or name of the remote host `sshfs` will mount, and that host's public key. A remote host often has multiple public keys. All of these keys should be added to the `known_hosts` file, each prepended with the host's IP address.

**NOTE** you can use hostname instead of IP at the start of a `known_hosts` line if you want. But whichever you use, **it must match what you use in the `fstab`**. In our `fstab.example`, we have `sshfs#myuser2@1.2.3.5:/Users/Shared/Legacy`. When constructing the consolidated `known_hosts` file, all lines for this remote host must therefore start with `1.2.3.5 `.

To get a given remote hosts' public keys, `ssh` to the remote host and look for files in `/etc/ssh/` with filenames matching the pattern `ssh_host_*key.pub`.

To construct a single `known_hosts` for accessing that host, `cat` all of the `/etc/ssh/ssh_host_*key.pub` files together into a single file and prepend every line with the host's IP address. With a bit of luck, the following command (run on the remote host to be **collect**ed from) will do this for you:

```
cat /etc/ssh/ssh_host_*key.pub | sed "s/^/`ipconfig getifaddr en0` /" >known_hosts
```

Your mileage may vary. Worst case, just use `cat /etc/ssh/ssh_host_*key.pub >known_hosts` and prepend each line manually with the IP of that host (taking care to match what's in `fstab`) using a text editor.

Once you've created a `known_hosts` file for each of the remote hosts to be backed up, `cat` together all of the `known_hosts` files into one almightly `known_hosts` file. Replace the empty `known_hosts` file supplied here with that consolidated file.

When you're done, a non-empty file called `known_hosts` must exist in the same directory as the `Dockerfile`.

### Build the docker image

```
docker build -t collect .
```

### (`sshfs` only) install the newly-built container's ssh public key on remote hosts

If you're not using `sshfs` in your `fstab` you can skip this step.

Assuming the build is successful, at some point during the build process, you will have seen output such as
```
...
---> Running in d4f5a0a30b4b
Copy to authorized_keys on hosts to collect from (if using sshfs):
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDBAooxLxSzH4Q9ButYFdV7N8BmP5SALALn3akX2ppoScsCJdwgzkM6INu0dkjdmMKOZsADy3SAe582ESvE0GR2l50f+UafUYS+nHBhtiNMfFImQ5awWfk+9o/OOgRlGNQSphomUQU4lXn50aPWO/vJZxqV5EbUBo0Hrh219N7MvZtm5e8pmJL8O7s747J6+SJ4xyZKze2TdtggRqntNCHkXlWrbUhaOte5n127HVkepHqeIHxhp6L9Z1WqZC0Bl89iYfssPBCI2PvRlCJ9vVWSthxbspvfjmxW+y9Ob9JIWGOy066/OHlBI2hyUyz7pvpkbIbROmj25B0H634HJfqZ1M6k3B91lSMdQzQCPJC6J1iyOiV0dVVWtoL/Vwv3KjgwiQCptS7LoV3F2dwiESJ2P4RjhybFP3FGOcMUDfTkBRfcXkbg0lsKWWvadnM817vN37XsIktpDkj05udTVooKHPQpatUwkfuJS/+2YcvvAl+XsJDcjVSzKOaj9sPrroM= root@d4f5a0a30b4b
Removing intermediate container d4f5a0a30b4b
...
```

Add the line starting `ssh-rsa` as a new line at the bottom of the `authorized_keys` files on all of the remote hosts that the container will sweep. For our example, we would add an extra line at the bottom of the file `~myuser2/.ssh/authorized_keys` on `1.2.3.5`.

## testing and troubleshooting

If all is well you should be able to start a new throw-away container of the newly-built **collect** image, and `mount` all of your `fstab` mounts.

```
$ docker run --rm -it --entrypoint /bin/bash --device /dev/fuse --cap-add SYS_ADMIN --security-opt apparmor:unconfined collect
root@ac583ef32b7c:~# cat /etc/fstab
# --------------------------------------------------------------------------------------------
# This is a regular Ubuntu fstab file to be used as the fstab inside the collect container.
# When try_collect mounts, Ubuntu will use this fstab file to perform the mount.
# The mountpoints listed here must match those in the sources file (see sources.example)
# --------------------------------------------------------------------------------------------
# sshfs devices should have authorized_keys installed on the source (e.g. at
# myuser2@1.2.3.5:~/.ssh/authorized_keys).  The string to add to the authorized_keys file
# is generated when building the collect container.  See README.md for more details.
#
# sshfs devices also must be known to the collect container when trying to mount.  This is
# accomplished by having a root@collect:~/.ssh/known_hosts file present inside the container.
# collect's Dockerfile here copies one in when building the collect container, but you must
# create it (see README.md).  Alternatively you can avoid this whole known_hosts thing by
# telling ssh not to bother checking the source's identity.  Adding ,StrictHostKeyChecking=no,
# to the list of options below will accomplish this.  This is insecure however.
# --------------------------------------------------------------------------------------------
# <device>                                  <mountpoint>            <type>  <options>
# --------------------------------------------------------------------------------------------
//1.2.3.4/Users                             /mnt/mymachine1_Users   cifs    auto,user,username=myuser1,password=mypassword1
sshfs#myuser2@1.2.3.5:/Users/Shared/Legacy  /mnt/mymachine2_legacy  fuse    auto,allow_other,user,reconnect
root@ac583ef32b7c:~# mkdir -v /mnt/mymachine1_Users
mkdir: created directory '/mnt/mymachine1_Users'
root@ac583ef32b7c:~# mkdir -v /mnt/mymachine2_legacy
mkdir: created directory '/mnt/mymachine2_legacy'
root@ac583ef32b7c:~# mount -a
root@ac583ef32b7c:~# ls /mnt/mymachine1_Users/
 john	paul	george	ringo
root@ac583ef32b7c:~# ls /mnt/mymachine2_legacy/
 Photos	Videos	Music	Docs	Stuff
root@ac583ef32b7c:~# exit
```

If you're using `sshfs`, you should be able to `ssh` to all of your remote hosts directly from this container without being prompted for anything.

## running

The following command will start the container and restart it on every NAS reboot. Run it from your NAS.

```
docker run -d --restart always --name collect -v /etc/localtime:/etc/localtime:ro -v /location/on/NAS/for/collected/files/:/backup -v /location/on/NAS/for/logs/:/logs --device /dev/fuse --cap-add SYS_ADMIN --security-opt apparmor:unconfined collect
```

`/location/on/NAS/for/collected/files/` must be a writable directory. The `rsync`s running inside **collect** will deposit files here.
This same location must be used for the **push** container, since it will grab files from here for `rclone`ing to the cloud.

A file will be created on your NAS called `/location/on/NAS/for/logs/collect.log`. You can `tail -f` it from the NAS to see what's going on.

Every ten minutes (as you'll see in `collect.log`), the **collect** container will ping all of the machines in `sources`; if any are alive, it will start an `rsync` to collect any new files which will end up in `/location/on/NAS/for/collected/files/`. It will only `rsync` once every day per source.

## stopping

If started as above, **collect** will restart on every NAS reboot. To kill it for good, use

```
docker kill collect
```
