---
title: Ansible简介及思想
date: 2017-12-21 18:27:44
layout: tag
tags: [Ansible, 设计哲学]
---

## ansible是什么

如果我们Google一下ansible，第一条出来的就是ansible的官网，它的title是“Ansible is Simple IT Automation”，从这里我们就能了解了ansible的目标：自动化。什么的自动化呢，其实是部署自动化(infrastructure as code)，将你原先一步一步使用命令转变为通过一系列的`状态检查`来安装一个软件，可以实现批量部署，一键部署。

## 为什么用ansible

ansible有很多的优势，我觉得最重要的就是简单。ansible无需你安装客户端，只需要在一台机器上安装好ansible，配置好ssh，就可以使用了。语法也很简单，使用一系列的task来指定要做的任务，yaml格式提供了很好的缩进，一目了然。

<!--more-->

## 如何使用ansible

ansible使用只需3步
- 控制机安装ansible，可以使用pip，yum或源码安装
- 在目标机上加入控制机的ssh pub key，在控制机上都ssh一下，将目标机加入到known-hosts中去
- 直接执行ansible命令或编写剧本来执行

ansible需要使用Python相关库，如果没有安装的话还需要安装，并且需要libselinux-python库（yum安装即可）。

## 编写剧本需要注意的事项

一定要编写可重复执行的剧本，也就是说playbook要是一系列对状态的定义，而不是一系列动作，在执行完后系统要达到什么样的状态，这样在重复执行剧本不会出什么问题。对应到开发上的定义，我们说编写的剧本执行需要具备幂等性，一次执行与多次执行结果一致。

譬如过说要安装一个rpm包，我们可以在playbook中写一个`shell: rpm -ivh a.rpm`,这是可以执行的，但是不符合ansible的哲学，因为当包已安装过后，再次执行就会报错。我们需要使用ansible提供的yum来定义状态
> \- name: Install package.
> &nbsp;   yum:
> &nbsp; &nbsp;      name: /tmp/package.rpm
> &nbsp; &nbsp;      state: present



## ansible的弊端

上面说了ansible简单、易上手，但同时我们也要了解它存在的问题才能决定是否适合我们。

- 性能，ansible部署速度比chef，puppet要慢一些，在大量机器上就会显现出来
- 成熟度，ansible界面、可用模块不如puppet成熟

## 替代性技术

- puppet
- chef
- saltstack
- Fabric

网上有很多相关的比较，各有优势，看大家的应用场景。

