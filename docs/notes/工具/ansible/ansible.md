# Ansible

## Inventory文件

Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 /etc/ansible/hosts

### 主机与组

/etc/ansible/hosts 文件的格式与windows的ini配置文件类似:

```
mail.example.com

[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com
three.example.com
```

方括号[]中是组名,用于对系统进行分类,便于对不同系统进行个别的管理.

一个系统可以属于不同的组,比如一台服务器可以同时属于 webserver组 和 dbserver组.这时属于两个组的变量都可以为这台主机所用,至于变量的优先级关系将于以后的章节中讨论.

如果有主机的SSH端口不是标准的22端口,可在主机名之后加上端口号,用冒号分隔.SSH 配置文件中列出的端口号不会在 paramiko 连接中使用,会在 openssh 连接中使用.

端口号不是默认设置时,可明确的表示为:

```
badwolf.example.com:5309
```

假设你有一些静态IP地址,希望设置一些别名,但不是在系统的 host 文件中设置,又或者你是通过隧道在连接,那么可以设置如下:

```
jumper ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50
```

在这个例子中,通过 “jumper” 别名,会连接 192.168.1.50:5555.记住,这是通过 inventory 文件的特性功能设置的变量. 一般而言,这不是设置变量(描述你的系统策略的变量)的最好方式.后面会说到这个问题.

一组相似的 hostname , 可简写如下:

```
[webservers]
www[01:50].example.com
```

数字的简写模式中,01:50 也可写为 1:50,意义相同.你还可以定义字母范围的简写模式:

```
[databases]
db-[a:f].example.com
```

对于每一个 host,你还可以选择连接类型和连接用户名:

对于每一个 host,你还可以选择连接类型和连接用户名:

```
[targets]

localhost              ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_ssh_user=mpdehaan
other2.example.com     ansible_connection=ssh        ansible_ssh_user=mdehaan
```

所有以上讨论的对于 inventory 文件的设置是一种速记法,后面我们会讨论如何将这些设置保存为 ‘host_vars’ 目录中的独立的文件.

### 主机变量

前面已经提到过,分配变量给主机很容易做到,这些变量定义后可在 playbooks 中使用:

```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```

### 组的变量

也可以定义属于整个组的变量:

```
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```

### 把一个组作为另一个组的子成员

可以把一个组作为另一个组的子成员,以及分配变量给整个组使用. 这些变量可以给 /usr/bin/ansible-playbook 使用,但不能给 /usr/bin/ansible 使用:

```
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[southeast:vars]
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```

如果你需要存储一个列表或hash值,或者更喜欢把 host 和 group 的变量分开配置,请看下一节的说明.

### 分文件定义Host和Group变量

在 inventory 主文件中保存所有的变量并不是最佳的方式.还可以保存在独立的文件中,这些独立文件与 inventory 文件保持关联. 不同于 inventory 文件(INI 格式),这些独立文件的格式为 YAML.

假设 inventory 文件的路径为:

```
/etc/ansible/hosts
```

假设有一个主机名为 ‘foosball’, 主机同时属于两个组,一个是 ‘raleigh’, 另一个是 ‘webservers’. 那么以下配置文件(YAML 格式)中的变量可以为 ‘foosball’ 主机所用.依次为 ‘raleigh’ 的组变量,’webservers’ 的组变量,’foosball’ 的主机变量:

```
/etc/ansible/group_vars/raleigh
/etc/ansible/group_vars/webservers
/etc/ansible/host_vars/foosball
```

举例来说,假设你有一些主机,属于不同的数据中心,并依次进行划分.每一个数据中心使用一些不同的服务器.比如 ntp 服务器, database 服务器等等. 那么 ‘raleigh’ 这个组的组变量定义在文件 ‘/etc/ansible/group_vars/raleigh’ 之中,可能类似这样:

```
---
ntp_server: acme.example.org
database_server: storage.example.org
```

这些定义变量的文件不是一定要存在,因为这是可选的特性.

还有更进一步的运用,你可以为一个主机,或一个组,创建一个目录,目录名就是主机名或组名.目录中的可以创建多个文件, 文件中的变量都会被读取为主机或组的变量.如下 ‘raleigh’ 组对应于 /etc/ansible/group_vars/raleigh/ 目录,其下有两个文件 db_settings 和 cluster_settings, 其中分别设置不同的变量:

```
/etc/ansible/group_vars/raleigh/db_settings
/etc/ansible/group_vars/raleigh/cluster_settings
```

### Inventory参数的说明

如同前面提到的,通过设置下面的参数,可以控制 ansible 与远程主机的交互方式,其中一些我们已经讲到过:

```
ansible_ssh_host
      将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.

ansible_ssh_port
      ssh端口号.如果不是默认的端口号,通过此变量设置.

ansible_ssh_user
      默认的 ssh 用户名

ansible_ssh_pass
      ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)

ansible_sudo_pass
      sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)

ansible_sudo_exe (new in version 1.8)
      sudo 命令路径(适用于1.8及以上版本)

ansible_connection
      与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.

ansible_ssh_private_key_file
      ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.

ansible_shell_type
      目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.

ansible_python_interpreter
      目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python
      不是 2.X 版本的 Python.我们不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 可执行程序名不可为 python以外的名字(实际有可能名为python26).

      与 ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....
```

一个主机文件的例子:

```
some_host         ansible_ssh_port=2222     ansible_ssh_user=manager
aws_host          ansible_ssh_private_key_file=/home/example/.ssh/aws.pem
freebsd_host      ansible_python_interpreter=/usr/local/bin/python
ruby_module_host  ansible_ruby_interpreter=/usr/bin/ruby.1.9.3
```

## Patterns

在Ansible中,Patterns 是指我们怎样确定由哪一台主机来管理. 意思就是与哪台主机进行交互. 

命令格式如下:

```
ansible <pattern_goes_here> -m <module_name> -a <arguments>
```

示例如下:

```
ansible webservers -m service -a "name=httpd state=restarted"
```

一个pattern通常关联到一系列组(主机的集合) –如上示例中,所有的主机均在 “webservers” 组中.

不管怎么样,在使用Ansible前,我们需事先告诉Ansible哪台机器将被执行. 能这样做的前提是需要预先定义唯一的 host names 或者 主机组.

如下的patterns等同于目标为仓库(inventory)中的所有机器:

```
all
*
```

也可以写IP地址或系列主机名:

```
one.example.com
one.example.com:two.example.com
192.168.1.50
192.168.1.*
```

如下patterns分别表示一个或多个groups.多组之间以冒号分隔表示或的关系.这意味着一个主机可以同时存在多个组:

```
webservers
webservers:dbservers
```

你也可以排队一个特定组,如下实例中,所有执行命令的机器必须隶属 webservers 组但同时不在 phoenix组:

```
webservers:!phoenix
```

你也可以指定两个组的交集,如下实例表示,执行命令有机器需要同时隶属于 webservers 和 staging 组.

> webservers:&staging

你也可以组合更复杂的条件:

```
webservers:dbservers:&staging:!phoenix
```

上面这个例子表示“‘webservers’ 和 ‘dbservers’ 两个组中隶属于 ‘staging’ 组并且不属于 ‘phoenix’ 组的机器才执行命令” ... 哟！唷! 好烧脑的说！

你也可以使用变量如果你希望通过传参指定group,ansible-playbook通过 “-e” 参数可以实现,但这种用法不常用:

```
webservers:!{{excluded}}:&{{required}}
```

你也可以不必严格定义groups,单个的host names, IPs , groups都支持通配符:

```
*.example.com
*.com
```

Ansible同时也支持通配和groups的混合使用:

```
one*.com:dbservers
```

在高级语法中,你也可以在group中选择对应编号的server:

```
webservers[0]
```

或者一个group中的一部分servers:

```
webservers[0-25]
```

大部分人都在patterns应用正则表达式,但你可以.只需要以 ‘~’ 开头即可:

```
~(web|db).*\.example\.com
```

同时让我们提前了解一些技能,除了如上,你也可以通过 `--limit` 标记来添加排除条件,/usr/bin/ansible or /usr/bin/ansible-playbook都支持:

```
ansible-playbook site.yml --limit datacenter2
```

如果你想从文件读取hosts,文件名以@为前缀即可.从Ansible 1.2开始支持该功能:

```
ansible-playbook site.yml --limit @retry_hosts.txt
```

## Introduction To Ad-Hoc Commands

在下面的例子中,我们将演示如何使用 /usr/bin/ansible 运行 ad hoc 任务.

所谓 ad-hoc 命令是什么呢?

(这其实是一个概念性的名字,是相对于写 Ansible playbook 来说的.类似于在命令行敲入shell命令和 写shell scripts两者之间的关系)...

如果我们敲入一些命令去比较快的完成一些事情,而不需要将这些执行的命令特别保存下来, 这样的命令就叫做 ad-hoc 命令.

Ansible提供两种方式去完成任务,一是 ad-hoc 命令,一是写 Ansible playbook.前者可以解决一些简单的任务, 后者解决较复杂的任务.

一般而言,在学习了 playbooks 之后,你才能体会到 Ansible 真正的强大之处在哪里.

那我们会在什么情境下去使用ad-hoc 命令呢?

比如说因为圣诞节要来了,想要把所有实验室的电源关闭,我们只需要执行一行命令 就可以达成这个任务,而不需要写 playbook 来做这个任务.

至于说做配置管理或部署这种事,还是要借助 playbook 来完成,即使用 ‘/usr/bin/ansible-playbook’ 这个命令.

### Parallelism and Shell Commands

举一个例子

这里我们要使用 Ansible 的命令行工具来重启 Atlanta 组中所有的 web 服务器,每次重启10个.

我们先设置 SSH-agent,将私钥纳入其管理:

```
$ ssh-agent bash
$ ssh-add ~/.ssh/id_rsa
```

如果不想使用 ssh-agent, 想通过密码验证的方式使用 SSH,可以在执行ansible命令时使用 `--ask-pass` (`-k`)选项, 但这里建议使用 ssh-agent.

现在执行如下命令,这个命令中,atlanta是一个组,这个组里面有很多服务器,”/sbin/reboot”命令会在atlanta组下 的所有机器上执行.这里ssh-agent会fork出10个子进程(bash),以并行的方式执行reboot命令.如前所说“每次重启10个” 即是以这种方式实现:

```
$ ansible atlanta -a "/sbin/reboot" -f 10
```

在执行 /usr/bin/ansible 时,默认是以当前用户的身份去执行这个命令.如果想以指定的用户执行 /usr/bin/ansible, 请添加 “-u username”选项,如下:

```
$ ansible atlanta -a "/usr/bin/foo" -u username
```

如果想通过 sudo 去执行命令,如下:

```
$ ansible atlanta -a "/usr/bin/foo" -u username --sudo [--ask-sudo-pass]
```

如果你不是以 passwordless 的模式执行 sudo,应加上 `--ask-sudo-pass` (`-K`)选项,加上之后会提示你输入 密码.使用 passwordless 模式的 sudo, 更容易实现自动化,但不要求一定要使用 passwordless sudo.

也可以通过``–sudo-user`` (`-U`)选项,使用 sudo 切换到其它用户身份,而不是 root(译者注:下面命令中好像写掉了–sudo):

```
$ ansible atlanta -a "/usr/bin/foo" -u username -U otheruser [--ask-sudo-pass]
```

>  在有些比较罕见的情况下,一些用户会受到安全规则的限制,使用 sudo 切换时只能运行指定的命令.这与 ansible的 no-bootstrapping 思想相悖,而且 ansible 有几百个模块,在这种限制下无法进行正常的工作. 所以执行 ansible 命令时,应使用一个没有受到这种限制的账号来执行.

以上是关于 ansible 的基础.如果你还没阅读过 patterns 和 groups,应先阅读 [Patterns](https://ansible-tran.readthedocs.io/en/latest/docs/intro_patterns.html) .

在前面写出的命令中, `-f 10` 选项表示使用10个并行的进程.这个选项也可以在 [Ansible的配置文件](https://ansible-tran.readthedocs.io/en/latest/docs/intro_configuration.html) 中设置, 在配置文件中指定的话,就不用在命令行中写出了.这个选项的默认值是 5,是比较小的.如果同时操作的主机数比较多的话, 可以调整到一个更大的值,只要不超出你系统的承受范围就没问题.如果主机数大于设置的并发进程数,Ansible会自行协调, 花得时间会更长一点.

ansible有许多模块,默认是 ‘command’,也就是命令模块,我们可以通过 `-m` 选项来指定不同的模块.在前面所示的例子中, 因为我们是要在 Atlanta 组下的服务器中执行 reboot 命令,所以就不需要显示的用这个选项指定 ‘command’ 模块,使用 默认设定就OK了.一会在其他例子中,我们会使用 `-m` 运行其他的模块,

### File Transfer

这是 /usr/bin/ansible 的另一种用法.Ansible 能够以并行的方式同时 SCP 大量的文件到多台机器. 命令如下:

```
$ ansible atlanta -m copy -a "src=/etc/hosts dest=/tmp/hosts"
```

若你使用 playbooks, 则可以利用 `template` 模块来做到更进一步的事情.

使用 `file` 模块可以做到修改文件的属主和权限,(在这里可替换为 `copy` 模块,是等效的):

```
$ ansible webservers -m file -a "dest=/srv/foo/a.txt mode=600"
$ ansible webservers -m file -a "dest=/srv/foo/b.txt mode=600 owner=mdehaan group=mdehaan"
```

使用 `file` 模块也可以创建目录,与执行 `mkdir -p` 效果类似:

```
$ ansible webservers -m file -a "dest=/path/to/c mode=755 owner=mdehaan group=mdehaan state=directory"
```

删除目录(递归的删除)和删除文件:

```
$ ansible webservers -m file -a "dest=/path/to/c state=absent"
```

### User and Group

使用 ‘user’ 模块可以方便的创建账户,删除账户,或是管理现有的账户:

```
$ ansible all -m user -a "name=foo password=<crypted password here>"

$ ansible all -m user -a "name=foo state=absent"
```

### Deploying From Source Control

直接使用 git 部署 webapp:

```
$ ansible webservers -m git -a "repo=git://foo.example.org/repo.git dest=/srv/myapp version=HEAD"
```

因为Ansible 模块可通知到 change handlers ,所以当源码被更新时,我们可以告知 Ansible 这个信息,并执行指定的任务, 比如直接通过 git 部署 Perl/Python/PHP/Ruby, 部署完成后重启 apache.

### Managing Services

确认某个服务在所有的webservers上都已经启动:

```
$ ansible webservers -m service -a "name=httpd state=started"
```

或是在所有的webservers上重启某个服务(译者注:可能是确认已重启的状态?):

```
$ ansible webservers -m service -a "name=httpd state=restarted"
```

确认某个服务已经停止:

```
$ ansible webservers -m service -a "name=httpd state=stopped"
```

### Time Limited Background Operations

需要长时间运行的命令可以放到后台去,在命令开始运行后我们也可以检查运行的状态.如果运行命令后,不想获取返回的信息, 可执行如下命令:

```
$ ansible all -B 3600 -P 0 -a "/usr/bin/long_running_operation --do-stuff"
```

如果你确定要在命令运行后检查运行的状态,可以使用 async_status 模块.前面执行后台命令后会返回一个 job id, 将这个 id 传给 async_status 模块:

```
$ ansible web1.example.com -m async_status -a "jid=488359678239.2844"
```

获取状态的命令如下:

```
$ ansible all -B 1800 -P 60 -a "/usr/bin/long_running_operation --do-stuff"
```

其中 `-B 1800` 表示最多运行30分钟, `-P 60` 表示每隔60秒获取一次状态信息.

Polling 获取状态信息的操作会在后台工作任务启动之后开始.若你希望所有的工作任务快速启动, `--forks` 这个选项的值 要设置得足够大,这是前面讲过的并发进程的个数.在运行指定的时间(由``-B``选项所指定)后,远程节点上的任务进程便会被终止.

一般你只能在把需要长时间运行的命令或是软件升级这样的任务放到后台去执行.对于 copy 模块来说,即使按照前面的示例想放到 后台执行文件传输,实际上并不会如你所愿.

### Gathering Facts

在 playbooks 中有对于 Facts 做描述,它代表的是一个系统中已发现的变量.These can be used to implement conditional execution of tasks but also just to get ad-hoc information about your system. 可通过如下方式查看所有的 facts:

```
$ ansible all -m setup
```

我们也可以对这个命令的输出做过滤,只输出特定的一些 facts,详情请参考 “setup” 模块的文档.

# Playbooks

Playbooks 与 adhoc 相比,是一种完全不同的运用 ansible 的方式,是非常之强大的.

简单来说,playbooks 是一种简单的配置管理系统与多机器部署系统的基础.与现有的其他系统有不同之处,且非常适合于复杂应用的部署.

Playbooks 可用于声明配置,更强大的地方在于,在 playbooks 中可以编排有序的执行过程,甚至于做到在多组机器间,来回有序的执行特别指定的步骤.并且可以同步或异步的发起任务.

我们使用 adhoc 时,主要是使用 /usr/bin/ansible 程序执行任务.而使用 playbooks 时,更多是将之放入源码控制之中,用之推送你的配置或是用于确认你的远程系统的配置是否符合配置规范.

对初学者,这里有一个 playbook,其中仅包含一个 play:

```
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: pkg=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

在下面,我们将分别讲解 playbook 语言的多个特性.

## playbook基础

### 主机与用户

你可以为 playbook 中的每一个 play,个别地选择操作的目标机器是哪些,以哪个用户身份去完成要执行的步骤（called tasks）.

hosts 行的内容是一个或多个组或主机的 patterns,以逗号为分隔符.

remote_user 就是账户名:

```
--- 
- hosts: webservers  
  remote_user: root
```

再者,在每一个 task 中,可以定义自己的远程用户:

```
---
- hosts: webservers
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: yourname
```

也支持从 sudo 执行命令:

```
---
- hosts: webservers
  remote_user: yourname
  sudo: yes
```

同样的,你可以仅在一个 task 中,使用 sudo 执行命令,而不是在整个 play 中使用 sudo:

```
---
- hosts: webservers
  remote_user: yourname
  tasks:
    - service: name=nginx state=started
      sudo: yes
```

你也可以登陆后,sudo 到不同的用户身份,而不是使用 root:

```
---
- hosts: webservers
  remote_user: yourname
  sudo: yes
  sudo_user: postgres
```

如果你需要在使用 sudo 时指定密码,可在运行 ansible-playbook 命令时加上选项 `--ask-sudo-pass` (-K). 如果使用 sudo 时,playbook 疑似被挂起,可能是在 sudo prompt 处被卡住,这时可执行 Control-C 杀死卡住的任务,再重新运行一次.

### Tasks列表

每一个 play 包含了一个 task 列表（任务列表）.一个 task 在其所对应的所有主机上（通过 host pattern 匹配的所有主机）执行完毕之后,下一个 task 才会执行.有一点需要明白的是（很重要）,在一个 play 之中,所有 hosts 会获取相同的任务指令,这是 play 的一个目的所在,也就是将一组选出的 hosts 映射到 task.（注:此处翻译未必准确,暂时保留原文）

在运行 playbook 时（从上到下执行）,如果一个 host 执行 task 失败,这个 host 将会从整个 playbook 的 rotation 中移除. 如果发生执行失败的情况,请修正 playbook 中的错误,然后重新执行即可.

每个 task 的目标在于执行一个 moudle, 通常是带有特定的参数来执行.在参数中可以使用变量（variables）.

modules 具有”幂等”性,意思是如果你再一次地执行 moudle（译者注:比如遇到远端系统被意外改动,需要恢复原状）,moudle 只会执行必要的改动,只会改变需要改变的地方.所以重复多次执行 playbook 也很安全.

对于 command module 和 shell module,重复执行 playbook,实际上是重复运行同样的命令.如果执行的命令类似于 ‘chmod’ 或者 ‘setsebool’ 这种命令,这没有任何问题.也可以使用一个叫做 ‘creates’ 的 flag 使得这两个 module 变得具有”幂等”特性 （不是必要的）.

每一个 task 必须有一个名称 name,这样在运行 playbook 时,从其输出的任务执行信息中可以很好的辨别出是属于哪一个 task 的. 如果没有定义 name,‘action’ 的值将会用作输出信息中标记特定的 task.

如果要声明一个 task,以前有一种格式: “action: module options” （可能在一些老的 playbooks 中还能见到）.现在推荐使用更常见的格式:”module: options” ,本文档使用的就是这种格式.

下面是一种基本的 task 的定义,service moudle 使用 key=value 格式的参数,这也是大多数 module 使用的参数格式:

```
tasks:
  - name: make sure apache is running
    service: name=httpd state=running
```

比较特别的两个 modudle 是 command 和 shell ,它们不使用 key=value 格式的参数,而是这样:

```
tasks:
  - name: disable selinux
    command: /sbin/setenforce 0
```

使用 command module 和 shell module 时,我们需要关心返回码信息,如果有一条命令,它的成功执行的返回码不是0, 你或许希望这样做:

```
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand || /bin/true
```

或者是这样:

```
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand
    ignore_errors: True
```

如果 action 行看起来太长,你可以使用 space（空格） 或者 indent（缩进） 隔开连续的一行:

```
tasks:
  - name: Copy ansible inventory file to client
    copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
            owner=root group=root mode=0644
```

在 action 行中可以使用变量.假设在 ‘vars’ 那里定义了一个变量 ‘vhost’ ,可以这样使用它:

```
tasks:
  - name: create a virtual host file for {{ vhost }}
    template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}
```

这些变量在 tempates 中也是可用的,稍后会讲到.

在一个基础的 playbook 中,所有的 task 都是在一个 play 中列出,稍后将介绍一种更合理的安排 task 的方式:使用 ‘include:’ 指令.

# Harbor搭建

```yaml
---
- hosts: harbor
  gather_facts: no
  vars:
    - compose_url: ./docker-compose 
    - harbor_name: harbor-offline-installer-v2.1.2.tgz
    - harbor_url: ./harbor-offline-installer-v2.1.2.tgz
    - harbor_yaml: ./harbor.yml
  tasks:
    - name: copy docker-compose
      copy: 
        src: "{{ compose_url }}"
        dest: /usr/local/bin/docker-compose
    - name: add exection
      shell: chmod +x /usr/local/bin/docker-compose
    - name: copy harbor
      copy:
        src: "{{ harbor_url }}"
        dest: /opt
    - name: unzip harbor
      shell: "tar -xvf /opt/{{ harbor_name }} -C /opt" 
    - name: rm harbor
      shell: "rm -rf /opt/{{ harbor_name }}"
    - name: copy yaml
      copy:
        src: "{{ harbor_yaml }}"
        dest: "/opt/harbor"
    - name: make data dir
      shell: "mkdir /data/harbor"
    - name: auth
      shell: "chmod 777 /data/harbor"
    - name: run harbor
      shell: "bash /opt/harbor/install.sh"
```

