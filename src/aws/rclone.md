# RClone

## Installing in the system

Install latest rclone from .db or .rpm packages. Be sure to have fuse3 installed, or install fuse and link ```ln -s /usr/bin/fusermount /usr/bin/fusermount3``` 

Put rclone.conf into /etc/rclone.conf

Put this into /etc/fstab

```
remote-name:bucket /mount-dir rclone rw,nofail,_netdev,x-systemd.automount,args2env,vfs_cache_mode=writes,vfs_refresh,allow_other,no_modtime,config=/etc/rclone.conf,cache_dir=/var/cache/rclone 0 0
```

## Installing rclone as a docker plugin

```sh
sudo mkdir -p /var/lib/docker-plugins/rclone/config
sudo mkdir -p /var/lib/docker-plugins/rclone/cache
```

```sh
docker plugin install rclone/docker-volume-rclone:amd64 args="-v" --alias rclone --grant-all-permissions
docker plugin list
```
Be sure to have rclone.conf in /var/lib/docker-plugins/rclone/config by default 
or you can set the rclone options during installation

```sh
docker plugin install rclone/docker-volume-rclone:amd64 --alias rclone --grant-all-permissions args="-v --allow-other --vfs-cache-mode=writes" config=/etc/rclone
```

or after that disabling the plugin

```sh
docker plugin disable rclone
docker plugin set rclone RCLONE_VERBOSE=2 config=/etc/rclone args="--vfs-cache-mode=writes --allow-other"
docker plugin enable rclone
docker plugin inspect rclone
```

### Docker-compose

```yml
volumes:
  volume_name_1:
    driver: rclone
    driver_opts:
      remote: 'remote-name:bucket'
      allow_other: 'true'
      vfs_cache_mode: full
      token: '{"type": "borrower", "expires": "2021-12-31"}'
      poll_interval: 0
```
