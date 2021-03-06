---
layout: post
title: ansible编写系统初始化playbook
date: 2016/03/24 21:30
categories: [ansible playbook]
tags: [系统初始化]
---

　　刚到公司时，从基础组同事那里拿到的虚拟主机的系统环境并不符合应用的要求，这时就需要应用运维的同事安装一些软件和配置一些环境变量，而这些事情往往是反复操作，极大的浪费时间。这里介绍使用ansible这个神器来解决系统初始化的操作。
<!--more-->

# 目的

　　系统初始化工作是一个简单、繁复的工作，从基础组得到的虚拟主机只是一个最最基础的系统环境，有许多安装包和环境所需要的配置都没有进行相应的设置，应用运维组的同事
需要针对虚拟主机担任角色不同而需要进行一此初始化工作，为了让应用组人员从完全人肉方式初始化系统中解脱出来，所以用ansible工具编写playbook来完成系统初始化工作。提
高系统初始化效率，减少主机上线时间，减少因人肉初始化系统带的操作失误。

# 主机角色及任务梳理

　　实际环境中因主机担任的角色不同而所需要初始化的操作也有所不同，需要整理出几个大类的主机角色，以及各种主机角色所在做的操作，关联到ansible上就所要做的就是task。

## 主机角色分类

<pre><code>1. 纯JDK运行环境
2. Tomcat运行环境
3. redis-cluster集群环境
4. Nginx环境
5. HAproxy环境
6. lvs环境
......
</code></pre>

## 主机所需执行task

　　根据主机所担任的角色不同所需要执行的task也有所不同，而有一些task是需要在所有主机执行。

- **基础task**

　　基础stsk是所有主机都需要执行的task，task整理如下：

<pre><code>1. 禁用ssh登陆DNS解析
2. 启用开机执行时间同步
3. 配置apt源
4. 安装基础软件包
5. 时间同步设置成crontab
6. 执行计划任务时启用日志功能
7. 设置主机网关（若无就设置网关）
8. 设置常用命令别名
9. 设置主机DNS
10. 修改主机openfile
11. 增加业务用户，设置密码，建立无密码验证
12. 修改root用户密码
......
</code></pre>

- **JDK运行环境所执行task**

 　　JDK环境只需要安装JDK包和配置必要的JAVA环境变量即可，针对JAVA的环境变量不会去编辑 `/etc/profile` 这个系统级的环境文件，只修改用户家目录下的 `.bashrc` 和 `.bash_profile` 文件。所执行task整理如下：
 
<pre><code>1. 拷贝JAVA二进制程序到远程主机的用户家目录
2. 拷贝.bashrc和.bash_profile文件到用户家目录
</code></pre>
 
 - **Tomcat运行环境执行task**
 
 　　Tomcat环境除了需要JDK外，还需要上传一个Tomcat包到远程主机，再配置Tomcat相应的环境变量，在配置环境变量时同样不会去改变系统级的环境变量，只是修改`.bashrc`和`.bash_profile`用户级的环境变量文件。需要执行的task整理如下：
 
 <pre><code>1. 拷贝JAVA二进制包到远程主机的用户家目录
2. 拷贝Tomcat二进制包到远程主机的用户家目录
3. 拷贝.bashrc和.bash_profile文件到用户家目录
4. 根据远程主机的内存大小来配置catalina.sh文件中的JVM的内存大小
......
</code></pre>
 
- **redis-cluster集群环境需执行task**

　　redis cluster集群基于3.0.5编译而成，redis cluster手动编译配置过程请参考[这里](http://zhaochj.blog.51cto.com/368705/1700892)。task整理如下：

<pre><code>1. 拷贝redis二进制包到远程主机
2. 生成redis.conf文件
3. 启动redis实例
... ...
</code></pre>

- **Nginx环境需执行task**

　　Nginx环境的部署采用自编译后的二进制包，编译过程请参考[这里](http://zhaochj.github.io/make-nginx-template/)，需要执行的task大致如下：

<pre><code>1. 安装依赖包
2. 把已编译好的二进制包拷贝到目标主机并解压到相应目录
3. 增加运行nginx的用户
4. 增加systemctl方式管理的nginx.service脚本
5. 提供样例配置文件
6. 配置nginx日志滚动
7. 提供常用的网络优化参数
8. 导出nginx的二进制文件
9. 开启nginx配置文件的语法高亮
......
</code></pre>


- **HAproxy环境所需执行task**

　　HAproxy的环境部署直接采用apt-get的安装，版本是1.5.8，需要执行的task整理如下：
<pre><code>1. 安装haproxy软件包
2. 提供样例配置文件
3. 开启haproxy配置文件的语法高亮
4. 开启haproxy的日志功能
......
</code></pre>

　　上边把生产环境中常见的环境罗列出来，下边以几个实际的例子说明ansible怎样把这些操作关联到一个个task上，这里恐怕不能涉及到所有的操作，只是列举一些常见的或者是容易出错的模块操作。

# 举例

　　ansible的所有操作都是基于模块，在使用ansible时一定要建立起这个概念，比如你要新建一个文件或目录是使用file模块，拷贝一个文件到远程主机可使用copy模块或synchronize模块，要执行一个命令是使用shell模块或raw模块等等。

## 基础task执行示例

　　以安装基础性的软件包为例，在上代码前先来说明一下ansible模块的目录组成结构，ansible把多个需要执行的task编排组织起来就组织成ansible的playbook，一个playbook的格式如下：
<pre><code>---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: name=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running (and enable it at boot)
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
</code></pre>
把上边的代码保存为一个以`.yml`或`.yaml`结尾的文件就可以完成软件安装等工作。以上代码来源[这里](http://docs.ansible.com/ansible/playbooks_intro.html)。

　　简单的编写一个playbook就能胜任许多工作，但andible的roles特性把playbook变得更为灵活，roles使ansible适应更为复杂的环境。要使用ansible的roles特性你得要遵守它的目录组成方式，类似如下：
<pre><code>ansible@ansible:~/playbooks/base/basic_setings_root$ tree .
.
├── hosts
├── roles
│   ├── add_route
│   │   └── tasks
│   │       └── main.yml
│   ├── chg_root_passwd
│   │   └── tasks
│   │       └── main.yml
│   ├── cron_service
│   │   └── tasks
│   │       └── main.yml
│   ├── cron_timesync
│   │   └── tasks
│   │       └── main.yml
│   └── install_pkg
│       └── tasks
│           └── main.yml
└── site.yml
</code></pre>
更详细关于anisble roles的说明请参考[这里](http://docs.ansible.com/ansible/playbooks_roles.html)

　　下边我们就编写一个roles来完成软件包的安装，操作如下：
<pre><code>ansible@ansible:/tmp/playbook/install_pkg$ pwd
/tmp/playbook/install_pkg
ansible@ansible:/tmp/playbook/install_pkg$ mkdir -pv roles/install_pkg/{task,files}
mkdir: created directory ‘roles’
mkdir: created directory ‘roles/install_pkg’
mkdir: created directory ‘roles/install_pkg/task’
mkdir: created directory ‘roles/install_pkg/files’
ansible@ansible:/tmp/playbook/install_pkg$ touch site.yml
ansible@ansible:/tmp/playbook/install_pkg$ ls
roles  site.yml
ansible@ansible:/tmp/playbook/install_pkg$ tree .
.
├── roles
│   └── install_pkg
│       ├── files
│       └── task
└── site.yml

4 directories, 1 file
ansible@ansible:/tmp/playbook/install_pkg$ vim roles/install_pkg/task/main.yml
---
- name: install some pkg
  apt: name={{ items }} state=present
  with_item:
     - vim
     - ntpdate
  tags: ins_pkg
ansible@ansible:/tmp/playbook/install_pkg$ vim site.yml
---
- hosts: server
  remote_user: root
  roles:
    - install_pkg

ansible@ansible:/tmp/playbook/install_pkg$ vim hosts  
[server]
172.31.11.71

# 执行以下命令就可以执行我们定义的task
ansible@ansible:/tmp/playbook/install_pkg$ ansible-playbook site.yml -i hosts -k
SSH password: 

PLAY ***************************************************************************

TASK [setup] *******************************************************************
ok: [172.31.11.71]

PLAY RECAP *********************************************************************
172.31.11.71               : ok=1    changed=0    unreachable=0    failed=0 

# 我这里的目标主机上已安装过vim和ntpdate两个软件包，所以输出状态changed是0  
</code></pre>

　　如果在使用ansible时不熟悉命令,可以用`ansible --help`查看ansible命令的帮助信息，用`ansible-playbook --help`查看ansible-playbook命令的帮助信息，用`ansible-doc module-name`来查看某个模块的使用帮助信息，`ansible-doc -l`可以列出可用的模块。善于使用这些命令可以让你更快熟悉ansible的使用。

## nginx部署

　　在[nginx模板制作](http://zhaochj.github.io/make-nginx-template/)中讲述了源码编译安装一个nginx并导入几个优秀的第三方模块，后期的批量部署会以此为模板，利用ansible工具部署nginx环境。

　　同样采用roles的方式来组织部署nginx环境的这个roles，目录结构如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx$ pwd
/tmp/playbook/dep_nginx
ansible@ansible:/tmp/playbook/dep_nginx$ ls
hosts  roles  site.yml
ansible@ansible:/tmp/playbook/dep_nginx$ tree .
.
├── hosts
├── roles
│   └── dep_nginx                     # 角色名称
│       ├── files
│       │   ├── nginx                 # 日志滚动时的配置文件
│       │   ├── nginx18.sh            # 导出nginx二进制时配置文件
│       │   ├── nginx18.tar.gz        # 打包后的二进制包
│       │   ├── nginx.service         # systemctl风格的启动脚本
│       │   └── sysctl.conf           # 网络优化参数配置文件
│       ├── tasks
│       │   └── main.yml              # 任务执行的主文件
│       └── vars
│           └── main.yml              # 存放变量的主文件
└── site.yml                          # ansible-playbook的入口文件

5 directories, 9 files
</code></pre>

nginx文件内容如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx/roles/dep_nginx/files$ cat nginx
/usr/local/nginx18/logs/*.log {
        daily
        missingok
        rotate 30
        compress
        delaycompress
        notifempty
        create 640 nginx staff
        sharedscripts
        postrotate
                [ -f /var/run/nginx18.pid ] && kill -USR1 `cat /var/run/nginx18.pid`
        endscript
}
</code></pre>

nginx18.sh文件如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx/roles/dep_nginx/files$ cat nginx18.sh 
export PATH=/usr/local/nginx18/sbin:$PATH
</code></pre>

nginx.service文件内容如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx/roles/dep_nginx/files$ cat nginx.service 
[Unit]
Description=nginx - high performance web server 
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
PIDFile=/var/run/nginx18.pid
ExecStartPre=/usr/local/nginx18/sbin/nginx -t -c /usr/local/nginx18/conf/nginx.conf
ExecStart=/usr/local/nginx18/sbin/nginx -c /usr/local/nginx18/conf/nginx.conf
ExecReload=/usr/local/nginx18/sbin/nginx -s reload
ExecStop=/usr/local/nginx18/sbin/nginx -s stop
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
</code></pre>

sysctl.conf文件增加的优化参数如下：

<pre><code># 调高系统的 IP 以及端口数据限制，从可以接受更多的连接
net.ipv4.ip_local_port_range = 1500 65400

#timewait 的数量
net.ipv4.tcp_max_tw_buckets = 10000

#关闭timewait 快速回收。TIME_WAIT状态的socket是否被快速回收是由tcp_tw_recycle和tcp_timestamps两个配置项共同决定的，
#tcp_timestamps默认一般就是开启的
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_tw_recycle = 0

#开启重用。允许将TIME-WAIT sockets 重新用于新的TCP 连接
net.ipv4.tcp_tw_reuse = 1

#开启SYN Cookies，当出现SYN 等待队列溢出时，启用cookies 来处理。
net.ipv4.tcp_syncookies = 1

#net.ipv4.tcp_max_syn_backlog参数决定了SYN_RECV状态队列的数量，一般默认值为512或者1024，即超过这个数量，系统将不再接受新的TCP连接请
#求，一定程度上可以防止系统资源耗尽。可根据情况增加该值以接受更多的连接请求。
#这个就是你说的tcp支持的队列数，tcp 连接超过这个队列长度，就不允许连接.
net.ipv4.tcp_max_syn_backlog = 10240

#web 应用中listen 函数的backlog 默认会给我们内核参数的net.core.somaxconn 限制到128，而nginx 定义的NGX_LISTEN_BACKLOG 默认为511，所以有必要调整这个值。
net.core.somaxconn = 2048

#当keepalive 起用的时候，TCP 发送keepalive 消息的频度。缺省是2小时
net.ipv4.tcp_keepalive_time = 30

#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。
net.ipv4.tcp_fin_timeout = 20
</code></pre>

tasks目录下的main.yml文件内容如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx/roles/dep_nginx$ cat tasks/main.yml

---
- name: 安装nginx依赖包
  apt: name={{ item }} state=present
  with_items:
       - openssl
       - libssl1.0.0
       - libssl-dev
       - zlib1g
       - libpcre3 
       - libpcre3-dev
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: depend_pkg

- name: copy nginx18.tar.gz to remote host
  synchronize: src=nginx18.tar.gz dest=/usr/local checksum=yes compress=yes perms=yes
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: copy_nginx18

- name: Unpack the nginx18.tar.gz
  shell: tar xf nginx18.tar.gz
  args:
    chdir: "{{ nginx_work_dir }}"
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: unpack_nginx

- name: 创建临时目录
  file: path=/var/tmp/nginx18/{{ item }} group=root owner=root mode=0755 state=directory
  with_items:
       - proxy
       - fastcgi
       - uwsgi
       - scgi
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: create_dir

- name: add nginx user
  user: name=nginx shell=/usr/sbin/nologin state=present
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: adduser_nginx

- name: export binary file
  copy: src=nginx18.sh dest=/etc/profile.d group=root owner=root mode=0644
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: export_binary

- name: systemctl way manage nginx
  copy: src=nginx.service dest=/lib/systemd/system group=root owner=root mode=0644
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: nginx.service

- name: nginx.conf syntax highlighting
  copy: src=.vim dest={{ ansible_env.HOME }} group=root owner=root mode=0644
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: nginx.vim

- name: 配置日志滚动，默认保留30天的日志备份
  copy: src=nginx dest=/etc/logrotate.d group=root owner=root mode=0644
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: log_rotate

- name: copy sysctl.conf,网络优化
  copy: src=sysctl.conf dest=/etc group=root owner=root mode=0644 backup=yes
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: copy_sysctl.conf

- name: exec sysctl -p
  shell: /sbin/sysctl -p
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: reload_sysctl.conf
  
- name: nginx service power up
  service: name=nginx.service state=started enabled=yes
  when: ansible_distribution == "Debian" and ansible_distribution_major_version == "8"
  tags: start_nginx
</code></pre>

vars目录下的main.yml文件内容如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx/roles/dep_nginx$ cat vars/main.yml

---
nginx_work_dir: /usr/local
</code></pre>

site.yml文件内容如下：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx$ cat site.yml

---
- hosts: server
  remote_user: root
  roles:
    
    - dep_nginx
</code></pre>

hosts文件的内容类似这样：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx$ cat hosts 
[server]
192.168.0.67
192.168.0.68
</code></pre>

hosts文件是的主机就是所要部署nginx环境的主。

运行这个以roles组织起的anisble-playbook，只需要运行如下命令，输入远程主机的root密码：

<pre><code>ansible@ansible:/tmp/playbook/dep_nginx$ ansible-playbook site.yml -i hosts -k 
SSH password: 
</code></pre>


# 总结

　　示例就列举上边两个，各种环境下的需求各样，不可能面面俱到。利用ansible的roles特性能完成许多任务，要想熟练使用ansible这个工具只有多练习，善于运用anisble的帮助信息。
