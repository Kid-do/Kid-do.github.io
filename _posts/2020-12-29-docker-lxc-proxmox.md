---
title: "Running Docker in an LXC Container on Proxmox"
header:
  image: /assets/images/2020-12-29-header.jpg
  caption: "Photo credit: [**Unsplash**](https://unsplash.com/photos/fBZOVyF-96w)"
  teaser: /assets/images/2020-12-29-header.jpg
categories:
  - blog
---

~~NOTE: a recent proxmox update has broken docker running in LXC, read [this](https://forum.proxmox.com/threads/docker-in-lxc-l%C3%A4uft-nicht-mehr.83651/) forum post for more information.~~ Now fixed

While it is possible to run Docker on the host OS, it makes more sense to run Docker inside a virtual machine or container. However, virtual machines have a higher overhead, so I wanted to try using an LXC container. By enabling two features, Docker will happily run inside an unprivileged LXC container.

Via command line, open the container's config file at `/etc/pve/lxc/xxx.conf` and add the line: 

```
features: keyctl=1,nesting=1
```

Or using the GUI, logged in as the root user:
![Proxmox GUI](/assets/images/2020-12-29-proxmox.jpg)

After installing Docker in the LXC container, Docker containers should be able to run without issue.
