---
title: "Saltstack Mine"
layout: post
category: translation
tags: [Salt Python]
excerpt: "许多初级到中级的的 PHP 程序员把 header() 函数当作某种神秘巫术. 他们可以照着代码示例把功能实现, 但是还是不知道到底它是如果运作的. 我最开始就是这样的.


实际上它非常简单. 在这篇文章中, 我会解释 HTTP 头(header) 是如何运作的, 它们与 PHP 的关系, 以及它们的 meta 标签 equivalents(对应物)"

---
###SALT MINE
Salt Mine用于收集来自minion任何数据并且将它保存在master上。然后该数据提供给所有的minions通过:`salt.modules.mine` 模块

这些数据在minion上收集并发回给master, master只维护最新的数据(长期的数据需要使用returners或扩展的job缓存).

#### MINE函数
开启Salt Mine的 `mine_functions`选项需要被应用到一个minion上，这个选项通过minion的配置文件被应用，或者minion的pillar。`mine_functions` 选项规定什么功能被执行并且哪些参数被传递。如果没有参数被传递，一个空列表必须被增加：

```
mine_functions:
  test.ping: []
  network.ip_addrs:
    interface: eth0
    cidr: '10.0.0.0/8'
```
##### MINE函数别名
函数别名可以用来提供有用的目的信息来允许同样的函数通过不同的参数多次调用

```
mine_functions:
  network.ip_addrs: [eth0]
  networkplus.internal_ip_addrs: []
  internal_ip_addrs:
    mine_function: network.ip_addrs
    cidr: 192.168.0.0/16
  loopback_ip_addrs:
    mine_function: network.ip_addrs
    lo: True
```

#### MINE更新间隔
Salt Mine函数在minion启动及预设的更新间隔进行执行. 默认的为每60分钟进行一次更新, 可以通过 mine_interval 选项进行调整:    
`/etc/salt/minion`:

```
mine_interval: 60
```

#### SALT-SSH中使用MINE
2015.5.0版本开始，salt-ssh支持`mine.get`.    
因为minion不能提供它们的`mine_functions`配置，我们在如下三个位置取回为指定的mine函数的参数，按照如下的顺序：
1. Roster data
2. Pillar
3. Master config

`mine_functions`被格式化就像个普通的salt一样，仅仅只是存储在一个不通的位置。    
下例是一个 roster的包含`mine_functions`例子:

```
test:
  host: 104.237.131.248
  user: root
  mine_functions:
    cmd.run: ['echo "hello!"']
    network.ip_addrs:
      interface: eth0
```

#### MINE的用法
. **mine.delete**:
> 删除minion上指定的函数内容，如果成功返回True    
> 例子：    
>     salt '*' mine.delete 'network.interfaces'    

![alt 结果](./mine.delete.png)
. **mine.flush**:
> 删除minion上所有的mine内容，如果成功返回True    
> 例子：    
>     salt '*' mine.flush    

. **mine.get**:
> 根据target, function和expr_form获取mine的数据
> target能通过`glob`, `prce`, `grain`, `grain_prce`进行匹配搜索    
> 例子:    
>     salt '*' mine.get '*' network.interfaces    
>     salt '*' mine.get 'os:CentOS' network.interfaces grain    

. **mine.send**:
> 发送一个特定的函数给mine    
> 例子：    
>     salt '*' mine.send network.interfaces eth0    

![alt result](./mine.send.png)
. **mine.update**:
> 执行配置好的函数并将数据发回给master. 这些要执行的函数会被master的配置文件，pillar和`function_cache`参数下的minion的配置文件中合并起来
> mine_functions:
>   network.ip_addrs:

#### 官网例子
在一个state里一个方法使用来自Salt Mine的数据。在sls文件里，通过jinja这些值可以被检索和使用。下面的例子是HAProxy配置文件的一部分，用web grain从所有minions拉取IP地址将它们添加到负载均衡的服务器池。    
`/srv/pillar/top.sls`:

```
base:
  'G@roles:web':
    - web
```
`/srv/pillar/web.sls`:

```
mine_functions:
  network.ip_addrs: [eth0]
```
`/etc/salt/minion.d/mine.conf`:

```
mine_interval: 5
```
`/srv/salt/haproxy.sls`

```
haproxy_config:
  file.managed:
    - name: /etc/haproxy/config
    - source: salt://haproxy_config
    - template: jinja
```
`/srv/salt/haproxy_config`:

```
<...file contents snipped...>

{% for server, addrs in salt['mine.get']('roles:web', 'network.ip_addrs', expr_form='grain').items() %}
server {{ server }} {{ addrs[0] }}:80 check
{% endfor %}

<...file contents snipped...>
```



