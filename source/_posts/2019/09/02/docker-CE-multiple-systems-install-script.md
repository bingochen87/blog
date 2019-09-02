---
title: Docker CE 多系统安装脚本
category: 技术
tags: ["Docker"]
summary: Docker-CE 多系统安装脚本
reward: true
---

Docker-CE的官方 install.sh 其实已经是过期了，在一些系统上，安装是有问题的，例如 `Cent OS` 及 `RHEL` （我知道 `RHEL` 应该是安装 Docker ee）。以下脚本可以支持多系统一个脚本搞定。

### 支持的系统
包括以下的，但对应的系统要求，请点击对应的系统查看。

* [Amazon Linux](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)
* [CentOS](https://docs.docker.com/install/linux/docker-ce/centos/#os-requirements)
* [Debian](https://docs.docker.com/install/linux/docker-ce/debian/#os-requirements)
* [Fedora](https://docs.docker.com/install/linux/docker-ce/fedora/#os-requirements)
* [RHEL(RedHat)](https://docs.docker.com/install/linux/docker-ee/rhel/#os-requirements)
* [Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/#os-requirements)

### 安装脚本
```bash
#!/bin/sh
set -e

get_distribution() {
	lsb_dist=""
	# Every system that we officially support has /etc/os-release
	if [ -r /etc/os-release ]; then
		lsb_dist="$(. /etc/os-release && echo "$ID")"
	fi
	# Returning an empty string here should be alright since the
	# case statements don't act unless you provide an actual value
	echo "$lsb_dist"
}

befor_install_docker() {
    user="$(id -un 2>/dev/null || true)"

	sh_c='sh -c'
	if [ "$user" != 'root' ]; then
		if command_exists sudo; then
			sh_c='sudo -E sh -c'
		elif command_exists su; then
			sh_c='su -c'
		else
			cat >&2 <<-'EOF'
			Error: this installer needs the ability to run commands as root.
			We are unable to find either "sudo" or "su" available to make this happen.
			EOF
			exit 1
		fi
	fi

	lsb_dist=$( get_distribution )
	lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"
	case "$lsb_dist" in

		centos)
            # add aliyun epel repo
            sudo cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo_back
            sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
            sudo yum makecache -y
            sudo yum install -y epel-release
		;;

	esac
}

install_docker() {
	lsb_dist=$( get_distribution )
	lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"
	case "$lsb_dist" in

		redhat|rhel)
            # https://docs.docker.com/install/linux/docker-ce/centos/
            sudo yum remove -y docker \
                     docker-client \
                     docker-client-latest \
                     docker-common \
                     docker-latest \
                     docker-latest-logrotate \
                     docker-logrotate \
                     docker-engine


            sudo yum install -y yum-utils \
                device-mapper-persistent-data \
                lvm2

            sudo yum-config-manager \
                --add-repo \
                https://download.docker.com/linux/centos/docker-ce.repo

            sudo yum install -y containerd.io docker-ce-18.09.1 docker-ce-cli-18.09.1
		;;

        amzn)
            sudo yum update -y
            sudo amazon-linux-extras install docker
        ;;

        *)
            curl -fsSL https://get.docker.com | bash
        ;;

	esac
}

after_install_docker() {
    sudo usermod -aG docker `logname`
	
    lsb_dist=$( get_distribution )
	lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"
	case "$lsb_dist" in

        # start docker for centos/fedora/redhat
		centos|fedora|redhat|rhel)
            sudo systemctl enable docker.service
            sudo systemctl start docker
		;;

        amzn)
            sudo service docker start
        ;;

	esac

	case "$lsb_dist" in

        # install iptable_nat for redhat
		redhat|rhel)
            sudo modprobe iptable_nat
            # persistent
            cat > /etc/sysconfig/modules/dockerd.modules <<- ENDEND
#!/bin/sh

exec /sbin/modprobe iptable_nat >/dev/null 2>&1
ENDEND

            sudo chmod +x /etc/sysconfig/modules/dockerd.modules
		;;

	esac
}

install() {
    befor_install_docker
    install_docker
    after_install_docker
}

install
```

### 关键的点

对于 Debian、Ubuntu、Amazon Linux、Fedora 这些系统没什么好说的，关键是 Cent OS 及 RHEL 这两个。

Cent OS 需要使用第三方源来安装，还不能用 `epel` 这个源，里面并没有 docker ce的东西。我用的是 `aliyun` 的。添加完源，就可以使用 docker 官方的安装脚本安装了。

对于 RHEL，可以直接按照 [docker-ce/centos](https://docs.docker.com/install/linux/docker-ce/centos/) 这个来安装。但是，安装完后，会发现网络有问题（具体是什么问题我就不贴出了），需要用以下方法处理才可以。
```bash
sudo modprobe iptable_nat
cat > /etc/sysconfig/modules/dockerd.modules <<- ENDEND
#!/bin/sh

exec /sbin/modprobe iptable_nat >/dev/null 2>&1
ENDEND

sudo chmod +x /etc/sysconfig/modules/dockerd.modules
```
