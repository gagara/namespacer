# namespacer

This repo contains set of scripts and instructions how to bootstrap network namespace (as well as mount namespace) for given user and start user's X-session in these namespaces.

It can be useful in case you need to have specific network configuration for each user in your system and have users logged in simultaneously. e.g. each user requires to be logged in in it's own VPN -- as such you need to have several VPN connections running in parallel in single WS.

## Namespace setup

[openrc/](openrc/) directory contains OpenRC scripts which can be used to bootstrap namespaces during boot for particular user. This can be done as:
```shell
# cp openrc/* /etc/local.d/

# ln -s /etc/local.d/20-ns{,-bob}.start
# ln -s /etc/local.d/20-ns{,-bob}.stop

# ln -s /etc/local.d/20-ns{,-alice}.start
# ln -s /etc/local.d/20-ns{,-alice}.stop
```
here we will create dedicated network and mount namespace for `bob` and `alice` users which should already exist in your system.

Following files will be created to track and "hold" respective namespaces:
* `/run/netns/bob`, `/run/mountns/bob` for user `bob`
* `/run/netns/alice`, `/run/mountns/alice` for user `alice`

## User X-session

In order to "place" user X-session into pre-created namespaces we need to start root X-session process in that namespaces. For Xfce you can do following:
1. Copy common `xinitrc` to user's home directory:
```shell
$ mkdir -p ~/.config/xfce4/
$ cp /etc/xdg/xfce4/xinitrc ~/.config/xfce4/
```
2. Patch it in the way that xfce4-session is started after we `nsenter`-ed necessary namespace. Something like:
```
exec sudo -E nsenter --net=/run/netns/$USER --mount=/run/mountns/$USER su $USER -c "xfce4-session"
```
You can find patched `xinitrc` in [xfce/](xfce/) directory.

> NB: most likely respective user needs to be added to sudoers. e.g.:
> ```shell
> # echo "bob     ALL = (root) NOPASSWD:SETENV: /usr/bin/nsenter --net=/run/netns/bob --mount=/run/mountns/bob su bob -c xfce4-session" >> /etc/sudoers.d/bob
> ```

# Tips/Tricks

## Docker

Of course if your whole X-session is running in one namespace but docker was started as a service in default namespace you can't use it as usual.

Workaround is to use dedicated docker service (as well as containerd) which should be started from your X-session after login.

In OpenRC-based Gentoo you can easily run several instances of docker and containerd by simply symlinking init scripts as follows:
```shell
# ln -s /etc/init.d/docker{,-bob}
# ln -s /etc/init.d/containerd{,-bob}

# ln -s /etc/init.d/docker{,-alice}
# ln -s /etc/init.d/containerd{,-alice}
```
but, you need to slightly patch init.d scripts to run it smoothly.

For docker you can use [this](docker/openrc/custom_containerd_svc.patch) user patch:
```shell
# mkdir -p /etc/portage/patches/app-containers/docker
# cp docker/openrc/custom_containerd_svc.patch /etc/portage/patches/app-containers/docker
# emerge -1a docker
```

For containerd you can use this [overlay](https://github.com/gagara/gentooverlay.git).
