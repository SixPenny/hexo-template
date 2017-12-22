---
title: Ansible模块
date: 2017-12-21 18:27:44
layout: tag
tags: [Ansible, 模块]
---


尽量使用ansible的模块，我们平常的shell命令不具备幂等性，譬如说创建一个文件夹，如果文件夹存在的话，命令执行不成功ansible就直接报错终止了。

使用ansible的模块可以指定系统的状态，在ansible执行完后只要系统达到指定的状态就可以了，文件夹如果存在就不做任何操作，不存在就创建，可以多次运行。

<!--more-->

## copy
用途：文件拷贝
选项：
backup：在覆盖之前将原文件备份，备份文件包含时间信息。有两个选项：yes|no
content：用于替代"src",可以直接设定指定文件的值
dest：必选项。要将源文件复制到的远程主机的绝对路径，如果源文件是一个目录，那么该路径也必须是个目录
directory_mode：递归的设定目录的权限，默认为系统默认权限match
force：如果目标主机包含该文件，但内容不同，如果设置为yes，则强制覆盖，如果为no，则只有当目标主机的目标位置不存在该文件时，才复制。默认为yes
others：所有的file模块里的选项都可以在这里使用
src：要复制到远程主机的文件在本地的地址，可以是绝对路径，也可以是相对路径。如果路径是一个目录，它将递归复制。在这种情况下，如果路径使用"/"来结尾，则只复制目录里的内容，如果没有使用"/"来结尾，则包含目录在内的整个内容全部复制，类似于rsync。match
follow	yes/no	当拷贝的文件夹内有link存在的时候，那么拷贝过match去的也会有link
force	yes/no	默认为yes,会覆盖远程的内容不一样的文件（可能文件名一样）。如果是no，就不会拷贝文件，如果远程有这个文件
group	设定一个群组拥有拷贝到远程节点的文件权限
mode  等同于chmod，参数可以为“u+rwx or u=rw,g=r,o=r”
owner	设定一个用户拥有拷贝到远程节点的文件权限


示例：
> \- name: 拷贝不覆盖
>  copy: src=/data/file dest=/data/file force=no

> \- name: 覆盖之前先备份一下
>  copy: src=/data/file dest=/data/file backup=yes


## file
用途：设定文件属性
可以用于创建文件夹，指定用户，指定权限等等

mode 等同于chmod，数字的话前面必须加0
owner 等同于chown，改变文件或文件夹所属用户
path  文件路径（必须）
recurse  递归应用文件夹底下的文件
src   链接的源文件，state只能为link
dest 链接目的文件
state  可选项有file link directory hard touch absent 

示例：
> \- name:创建文件夹
>  \- file: path=/data/dir1 state=directory

> \- name 创建一个文件
>  \- file:
> &nbsp;  &nbsp; path: /etc/foo.conf
> &nbsp;  &nbsp; state: touch
> &nbsp;  &nbsp;  mode: "u+rw,g-wx,o-rwx"

> \- name 创建多个
> \- file:
> &nbsp;  &nbsp;     src: '/tmp/{{ item.src }}'
> &nbsp;  &nbsp;     dest: '{{ item.dest }}'
> &nbsp;  &nbsp;     state: link
> &nbsp;  &nbsp;    with_items:
> &nbsp;  &nbsp;  &nbsp;  &nbsp;    - { src: 'x', dest: 'y' }
> &nbsp;  &nbsp;    &nbsp;  &nbsp;  - { src: 'z', dest: 'k' }


## yum
用途：安装软件包，包括rpm包
ps:不建议使用rpm -ivh来安装rpm包，这样脚本只能执行一次
选项：
name：安装的软件包名
state: 软件状态 安装present installed latest 移除 absent removed


示例：
> \- name: Install package.
> &nbsp;   yum:
> &nbsp; &nbsp;      name: /tmp/package.rpm
> &nbsp; &nbsp;      state: present
 
 >\- name: remove the Apache package
  > &nbsp;  yum:
  >   &nbsp;  &nbsp;  name: httpd
  >   &nbsp;  &nbsp;  state: absent


## 最后

ansible提供了非常多的模块，可以参考官方文档来使用
所有模块的地址：http://docs.ansible.com/ansible/latest/list_of_all_modules.html