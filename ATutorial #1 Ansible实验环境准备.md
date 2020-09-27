Ansible是一个自动化运维框架，由Python语言开发，通过ssh实现无`Agent`对服务器进行一些列的自动化管理，比如进行软件安装、配置文件更新、文件分发等操作。这些功能的实现实际上是通过Ansible的诸多模块实现的，通过与模块之间的交互通信，实现这些功能。今天我们首先准备一下Ansible的实验环境，然后在此试验环境内进行Ansible由浅入深的学习。

通过轻量化的容器充当虚拟机,作为Ansible实验学习的基础环境,因此我们需要配置一个可以带有SSHD服务的容器，注意Dockerfile中登录容器的账号和密码为`root:password`
```Dockerfile
FROM ubuntu:18.04
RUN sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list

RUN apt-get update

RUN apt-get install -y openssh-server
RUN mkdir /var/run/sshd

RUN echo 'root:password' |chpasswd


RUN sed -ri 's/session required pam_loginuid.so/#session required pam_loginuid.so/g' /etc/pam.d/sshd
RUN sed -ri 's/^#?PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config

RUN mkdir /root/.ssh

RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

EXPOSE 22

CMD    ["/usr/sbin/sshd", "-D"]
```


构建镜像
```bash
docker build -t ansible_vm:v1 -f Dockerfile .
```

然后批量运行多个容器，初始化"虚拟机"环境:
```bash
root@nodec:~/workspace/ansible# for i in `seq 1 5`;do docker run -d --name ansible_vm_$i ansible_vm:v1;done
6cff77c574620987439c07a4b2698cd8bbff5ef46def505877df9bfd6b9a9966
cedb69ba3a1767116db2fc19ba64571f5e4e7d4269ad668881ef57d99b89c0e8
54278e64f06883b9d726a1c8d1458c53610c7375e345846183e36a59dee30040
921b7ec45b6cfd8569f25ac222525423b35c2272321c6585bc96c0e9839ea97d
beb134faa1098a107e742d0c9c81a542af9ea65ccee14ee5f66ce3250768c202
root@nodec:~/workspace/ansible# docker ps -a |grep ansible_vm
beb134faa109        ansible_vm:v1                               "/usr/sbin/sshd -D"      14 seconds ago      Up 11 seconds                     22/tcp                              ansible_vm_5
921b7ec45b6c        ansible_vm:v1                               "/usr/sbin/sshd -D"      16 seconds ago      Up 13 seconds                     22/tcp                              ansible_vm_4
54278e64f068        ansible_vm:v1                               "/usr/sbin/sshd -D"      18 seconds ago      Up 15 seconds                     22/tcp                              ansible_vm_3
cedb69ba3a17        ansible_vm:v1                               "/usr/sbin/sshd -D"      21 seconds ago      Up 18 seconds                     22/tcp                              ansible_vm_2
6cff77c57462        ansible_vm:v1                               "/usr/sbin/sshd -D"      23 seconds ago      Up 20 seconds                     22/tcp                              ansible_vm_1

root@nodec:~/workspace/ansible# for i in `seq 1 5`;do docker inspect ansible_vm_$i -f {{.NetworkSettings.Networks.bridge.IPAddress}};done > ansible_vm_ips
root@nodec:~/workspace/ansible# cat ansible_vm_ips # 所有测试节点的IP
172.17.0.2
172.17.0.3
172.17.0.4
172.17.0.5
172.17.0.6

# 如果需要销毁这些容器,参考下方的命令👹
for i in `seq 1 5`;do docker stop ansible_vm_$i && docker rm ansible_vm_$i;done
```

节点准备完成之后，我们准备在宿主机上开始安装Ansible

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
date;ansible --version
Sat Sep 26 18:25:53 CST 2020
ansible 2.9.13
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.12 (default, Jul 21 2020, 15:19:50) [GCC 5.4.0 20160609]
```

准备由ansible托管的机器清单,这里我们直接修改前面我们通过docker准备的ip列表文件
```bash
root@nodec:~/workspace/ansible# sed -i '1 i[docker]' ansible_vm_ips 
root@nodec:~/workspace/ansible# cat ansible_vm_ips 
[docker]
172.17.0.2
172.17.0.3
172.17.0.4
172.17.0.5
172.17.0.6
# Ansible官方把由ansible托管的机器列表配置文件叫做inventory.cfg. 所以我们重命名一下
root@nodec:~/workspace/ansible# mv ansible_vm_ips inventory.cfg
```

最后一步重要的步骤就是配置无密访问这些托管的机器
```bash
root@nodec:~/workspace/ansible# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:7mxkv8t2FXzVS43pOwCQ3fT4THravv1oFKSMTXXKjVc root@nodec.devopsman.cn
The key's randomart image is:
+---[RSA 2048]----+
|        .+ o...oE|
|        . o o++*=|
|           *.=B =|
|          . ===o.|
|        S   ..+= |
|       .o    +=  |
|       o..  .o.. |
|       o..o ...o |
|       .o.++ .+.+|
+----[SHA256]-----+

# 然后将公钥分发给这些托管机器
root@nodec:~/workspace/ansible# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.2
root@nodec:~/workspace/ansible# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.3
root@nodec:~/workspace/ansible# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.4
root@nodec:~/workspace/ansible# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.5
root@nodec:~/workspace/ansible# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.17.0.6
```

现在我们通过ansible的基础模块`ping`进行测试
```bash
root@nodec:~/workspace/ansible# ansible docker -i ./inventory.cfg -m ping
172.17.0.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
172.17.0.5 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
172.17.0.6 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
172.17.0.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
172.17.0.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

这样我们的Ansible实验环境就完成了。
