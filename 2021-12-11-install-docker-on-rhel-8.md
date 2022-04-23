---
title: Install Docker on RHEL 8
tags: docker rhel linux instructional
excerpt: "This is an instructional article that provides a simple methodology on how to install Docker onto Red Hat Enterprise Linux 8" 
---
## Introduction
This instructional documents the process to install Docker onto Red Hat Enterprise Linux 8. Note that we do not use Red Hat repositories for this, but rather we add Red Hat repositories via `yum-config-manager`, and manually edit `/etc/yum.repos.d/docker-ce.repo` to use CentOS 8 repositories instead. This is because there are no binaries built for x86_64 architecture within the Red Hat repositories. This methodology has always allowed myself to run the newer versions of container.io, and I've never had an issue with a deployment using this method, so I thought I should share it.  

## Installation
1. Install `yum-utils`
        ```bash
        sudo yum install -y yum-utils
        ```
  2. Add RHEL Docker Repo
        ```bash
        sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/rhel/docker-ce.repo`
        ```
3. Edit `/etc/yum.repos.d/docker-ce.repo` to use CentOS repositories rather then RHEL  [^1]
        ```bash
        sudo vim /etc/yum.repos.d/docker-ce.repo
        ```
        1. Enter `:`, then enter `%s/rhel/centos/g`, then press Return on your keyboard
        2. Save the file
                Enter `:x` 
4. Install Docker [^2]
        ```bash
        sudo yum install docker-ce -y
        ```
5. Enable & Start Docker
        ```bash
        sudo system enable docker --now
        ```

## Conclusion

The previous process should have now left you with an operational docker system, ready for ad-hoc control via `docker` commands. 

---

[^1]:   This may appear a little sketchy as we adding the RHEL Docker Repository to yum.repos.d, and then editing the Docker repository configuration to actually use the CentOS Docker repository, however it's the simplest and most reliable installation method I've employed since using RHEL 8.

[^2] You can also added any needed tools, e.g docker-compose, to this command. 