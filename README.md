**Название домашнего задания:**

"Обновить ядро в базовой системе"

**Текст задания:**

- Для выполнения домашнего задания используйте методичку
https://docs.google.com/document/d/12sC884LuLGST3-tZYQBDPvn6AH8AGJCK/edit?usp=share_link&ouid=107126378526912727172&rtpof=true&sd=true
- Выполните действия, описанные в методичке.
- Полученный в ходе выполнения ДЗ Vagrantfile, который будет разворачивать виртуальную машину используя vagrant box который вы загрузили Vagrant Cloud, залейте в ваш git-репозиторий.

**Описание команд и их вывод**

1. apt install packer vagrant virtualbox-6.1
2. mkdir /test_vm/
3. mkdir /test_vm/packer
4. mkdir /test_vm/packer/http
5. mkdir /test_vm/packer/scripts
6. cd /test_vm/packer && vi ./centos.json
```
{
"builders": [
    {
      "boot_command": [
        "<tab> inst.text inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg<enter><wait>"
      ],
      "boot_wait": "5s",
      "disk_size": "15240",
      "export_opts": [
        "--manifest",
        "--vsys",
        "0",
        "--description",
        "{{user `artifact_description`}}",
        "--version",
        "{{user `artifact_version`}}"
      ],
      "guest_os_type": "RedHat_64",
      "http_directory": "http",
      "iso_checksum": "b4bb35e2c074b4b9710419a9baa4283ce4a02f27d5b81bb8a714b576e5c2df7a",
      "iso_url": "http://mirrors.datahouse.ru/centos/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-latest-boot.iso",
      "name": "{{user `image_name`}}",
      "output_directory": "builds",
      "shutdown_command": "echo 'vagrant' | sudo -S /sbin/halt -h -p",
      "shutdown_timeout": "5m",
      "ssh_password": "vagrant",
      "ssh_port": 22,
      "ssh_pty": true,
      "ssh_timeout": "300m",
      "ssh_username": "vagrant",
      "type": "virtualbox-iso",
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "2048"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "4"
        ]
      ],
      "vm_name": "packer-centos-vm1"
    }
  ],
  "post-processors": [
    {
       "compression_level": "7",
      "output": "centos-{{user `artifact_version`}}-kernel-6-x86_64-Minimal.box",
      "type": "vagrant"
    }
  ],

  "provisioners": [
    {
      "execute_command": "{{.Vars}} echo 'vagrant' | sudo -S -E bash '{{.Path}}'",
      "expect_disconnect": true,
      "override": {
        "{{user `image_name`}}": {
          "scripts": [
            "scripts/stage-1-kernel-update.sh",
            "scripts/stage-2-clean.sh"
          ]
        }
      },
      "pause_before": "60s",
      "start_retry_timeout": "3m",
      "type": "shell"
    }
  ],
  "variables": {
    "artifact_description": "CentOS Stream 8 with kernel 6.x",
    "artifact_version": "8",
    "image_name": "centos-8"
  }
}
```
7. vi ./http/ks.cfg
```
eula --agreed

lang en_US.UTF-8
keyboard us
timezone UTC+3

network --bootproto=dhcp --device=link --activate
network --hostname=otus-c8

rootpw vagrant
authconfig --enableshadow --passalgo=sha512
user --groups=wheel --name=vagrant --password=vagrant --gecos="vagrant"

selinux --enforcing

firewall --disabled

firstboot --disable

text
url --url="http://mirror.linux-ia64.org/centos/8-stream/BaseOS/x86_64/os"

bootloader --location=mbr --append="ipv6.disable=1 crashkernel=auto"

skipx
logging --level=info
zerombr
clearpart --all --initlabel
autopart --type=lvm
reboot

%packages --ignoremissing
@^minimal-environment
%end
```
8. vi ./scripts/stage-1-kernel-update.sh
```
#!/bin/bash -x

sudo yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm 
yum --enablerepo elrepo-kernel install kernel-ml -y
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-set-default 0
mkdir /vagrant
chmod 755 /vagrant
echo "vagrant ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
echo "Grub update done."
echo "Reboot."
shutdown -r now
```
9. vi ./scripts/stage-2-clean.sh
```
#!/bin/bash -x

yum update -y
yum clean all

mkdir -pm 700 /home/vagrant/.ssh
curl -sL https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -o /home/vagrant/.ssh/authorized_keys
chmod 0600 /home/vagrant/.ssh/authorized_keys
chown -R vagrant:vagrant /home/vagrant/.ssh

rm -rf /tmp/*
rm  -f /var/log/wtmp /var/log/btmp
rm -rf /var/cache/* /usr/share/doc/*
rm -rf /var/cache/yum
rm -rf /vagrant/home/*.iso
rm  -f ~/.bash_history
history -c

rm -rf /run/log/journal/*
sync
grub2-set-default 0
echo "###   Hi from second stage" >> /boot/grub2/grub.cfg
```
10. packer build centos.json
```
==> Builds finished. The artifacts of successful builds are:
--> centos-8: 'virtualbox' provider box: centos-8-kernel-6-x86_64-Minimal.box 
```
11. * *Далее ввиду отсутсвия рабочего VPN, подлючился к https://app.vagrantup.com/HellolightSP/boxes/ через VPN плагин в браузере Chrome, авторизовался используя аккаунт Sign in with GitHub, создал New Vagrant box "centos-8-kernel-6" Box version 1.0. В Edit Provider указал:* *
```
Provider      -> virtualbox
Checksum type -> SHA256
Checksum      -> b4bb35e2c074b4b9710419a9bafc4283ce4a02b27d5b81bb8a714b576e5c2df7a
Upload File   -> загрузил свой созданный образ centos-8-kernel-6-x86_64-Minimal.box
```
12. vagrant init HellolightSP/centos-8-kernel-6
13. vagrant box add --name 'HellolightSP/centos-8-kernel-6' /home/neon/Downloads/centos-8-kernel-6-x86_64-Minimal.box
14. vagrant box list
15. vi Vagrantfile    * *# редактируем параметры согласно методичке https://docs.google.com/document/d/12sC884LuLGST3-tZYQBDPvn6AH8AGJCK/edit?usp=share_link&ouid=107126378526912727172&rtpof=true&sd=true* *
16. vagrant up
17. vagrant ssh
18. git clone https://github.com/hellolightSP/otus.git
19. cd ./otus
20. cp /test_vm/Vagrantfile ./
21. git add --all
22. git config --global user.email "hellolight2011@gmail.com"
23. git commit -m "add Vagrant file"
24. git push origin   * *#вводим логин и вместо пароля нужно в git создать токен, т.к. "Support for password authentication was removed on August 13, 2021."* *
