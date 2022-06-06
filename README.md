
## 自动化工具方案
在运维工作中经常会有一些需要"执行"来完成的操作，比如我们需要在目标机器上开设一个用户、安装配置一个服务以及完成一个服务的上下线等，而完成这种操作的方式有很多，比如我们可以手动执行一个命令、封装所有命令操作至一个执行脚本然后手动执行脚本。而实际工作中常常会涉及到多台机器的批量操作，运维同学大概率不会去手动的在对应机器上一次次的执行任务脚本，通常会借助一些所谓的自动化工具来完成，例如 shell 的 pssh 命令工具、python 的 fabric 等。对所有操作能够进行有效封装且保证效率与便捷性就显得尤为作用，一个好的执行工具不仅能提升大批量执行的效率也会减少操作的失误率进而很好的完善整个自动化体系，最终实现提升运维效率的目的。
 
业内此类的工具有很多，也有公司会根据自己的需求进行自研，本篇主要介绍 [ansible](https://www.ansible.com/) 和 [saltstack](https://saltproject.io/)，对比二者在不同业务场景中的实践表现。
## Ansible 与 saltstack 简单对比
鉴于运维实际生产环境中运维同学对二者的使用兼而有之，所以不会对二者的某些特性例如：安全性、运行原理、部署便捷性等进行介绍对比，下面只简单的列出一些常规的特性对比(此处引用网上的一篇评测数据)

| 对比 | SaltStack | Ansible |
| ---- |  ----  | ---- |
| 速度  | 依靠 Zeromq(默认), 速度快  | 基于 ssh 速度较慢
| 安全  | AES 加密 | SSH 安全
| 运行架构 | C/S 为主 | SSH
| 学习曲线 | 较为陡峭 | 相对容易
| 执行状态 | state、Formula | playbook、Role
| 开发语言 | python | playbook
| 架构扩展 | Master/Minion/Syndic | 无明显方案
| 自定义模块 | 支持 | 支持

__在以下业务实践部分会对部分涉及特性进行介绍说明__
 
## 业务场景实践
### 前提
#### 版本
Ansible
```shell
# ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Mar 15 2022, 12:22:08) [GCC 9.4.0]
```
Saltstack
```shell
# salt --version
salt 3004.1
```
#### 执行方式
Ansible 的执行以 playbook 为主，Salt 则以 state 状态为主
### 一些特性说明
#### 执行顺序依赖
通常我们在写 shell 脚本时，都会做上下执行功能代码的返回值判断，比如创建一个用户 zhangsan，在创建成功的情况下为用户创建密码(这里不深究 shell 的执行异常退出，例如：set -x)。这样能更好的对执行业务逻辑进行控制，如若不然则可能会造成整个执行过程不可逆的"错误"后果（笔者曾经经历过60游戏服合1的场景，因为执行过程没有做好上下依赖判断，执行中途错误而没有立即停止，导致超过合服涉及数几倍的服受影响，最后手动修复也是花费了许久的时间，对业务影响极大，当然这里面也有操作幂等的问题，这部分后续会提到），所以对执行过程的精确控制显得尤为重要。
 
Ansible playbook 每个 task 顺序执行会默认保持上下依赖（当然需要也只可对 task 做依赖控制）
```shell
# cat t1.yaml
- name: t1 playbook
  hosts: dev
  remote_user: root
  gather_facts: True
  tasks:
    - name: task 1
      shell: "exit 99"
    - name: task 2
      debug:
        msg: "task 2.."
 
# ansible-playbook t1.yaml
PLAY [t1 playbook] *********************************************************************************************************************************************
TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [192.168.1.80]
ok: [192.168.1.81]
TASK [task 1] **************************************************************************************************************************************************
fatal: [192.168.1.80]: FAILED! => {"changed": true, "cmd": "exit 99", "delta": "0:00:00.034632", "end": "2022-05-12 17:19:37.237382", "msg": "non-zero return code", "rc": 99, "start": "2022-05-12 17:19:37.202750", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
fatal: [192.168.1.81]: FAILED! => {"changed": true, "cmd": "exit 99", "delta": "0:00:00.035789", "end": "2022-05-12 17:19:37.236581", "msg": "non-zero return code", "rc": 99, "start": "2022-05-12 17:19:37.200792", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.80               : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
192.168.1.81               : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```
可见当 task 1 失败时，整个任务会直接终止掉而不会继续执行下去（如果需要对 task 1 做错误忽略处理，可使用 ignore_errors: True 选项来约束）
 
Salt state 每个 task 顺序执行默认则不会保持上下依赖，即task 1 执行失败， task 2 也会执行
```shell
# cat t1.sls
task 1:
  cmd.run:
    - name: "exit 99"
task 2:
  cmd.run:
    - name: "echo 'task 2'"


# salt -N group3 state.apply t1
192.168.1.80:
----------
          ID: task 1
    Function: cmd.run
        Name: exit 99
      Result: False
     Comment: Command "exit 99" run
     Started: 17:58:41.793724
    Duration: 13.428 ms
     Changes:
              ----------
              pid:
                  3380
              retcode:
                  99
              stderr:
              stdout:
----------
          ID: task 2
    Function: cmd.run
        Name: echo 'task 2'
      Result: True
     Comment: Command "echo 'task 2'" run
     Started: 17:58:41.807780
    Duration: 10.079 ms
     Changes:
              ----------
              pid:
                  3381
              retcode:
                  0
              stderr:
              stdout:
                  task 2
Summary for 192.168.1.80
------------
Succeeded: 1 (changed=2)
Failed:    1
------------
Total states run:     2
Total run time:  23.507 ms
192.168.1.81:
----------
          ID: task 1
    Function: cmd.run
        Name: exit 99
      Result: False
     Comment: Command "exit 99" run
     Started: 17:58:41.797014
    Duration: 14.766 ms
     Changes:
              ----------
              pid:
                  3240
              retcode:
                  99
              stderr:
              stdout:
----------
          ID: task 2
    Function: cmd.run
        Name: echo 'task 2'
      Result: True
     Comment: Command "echo 'task 2'" run
     Started: 17:58:41.812195
    Duration: 11.124 ms
     Changes:
              ----------
              pid:
                  3241
              retcode:
                  0
              stderr:
              stdout:
                  task 2
Summary for 192.168.1.81
------------
Succeeded: 1 (changed=2)
Failed:    1
------------
Total states run:     2
Total run time:  25.890 ms
ERROR: Minions returned with non-zero exit code
```
通过结果可知 task 1 返回错误时 task 2 任然执行，如果想要达到类似 ansible 的效果，可以使用 failhard 关键字约束
```shell
# cat t1.sls
task 1:
  cmd.run:
    - name: "exit 99"
    - failhard: True
task 2:
  cmd.run:
    - name: "echo 'task 2'"


# salt -N group3 state.apply t1
192.168.1.80:
----------
          ID: task 1
    Function: cmd.run
        Name: exit 99
      Result: False
     Comment: Command "exit 99" run
     Started: 18:03:35.048239
    Duration: 15.796 ms
     Changes:
              ----------
              pid:
                  3436
              retcode:
                  99
              stderr:
              stdout:
Summary for 192.168.1.80
------------
Succeeded: 0 (changed=1)
Failed:    1
------------
Total states run:     1
Total run time:  15.796 ms
192.168.1.81:
----------
          ID: task 1
    Function: cmd.run
        Name: exit 99
      Result: False
     Comment: Command "exit 99" run
     Started: 18:03:35.077703
    Duration: 18.398 ms
     Changes:
              ----------
              pid:
                  3296
              retcode:
                  99
              stderr:
              stdout:
Summary for 192.168.1.81
------------
Succeeded: 0 (changed=1)
Failed:    1
------------
Total states run:     1
Total run time:  18.398 ms
ERROR: Minions returned with non-zero exit code
```
也可通过全局配置 failhard 使得默认执行为
```shell
# cat /etc/salt/master
...
failhard: True
...
```
__小结__
编写 playbook 或 state 时应当注意执行上下约束的默认行为，可以根据实际场景需求来更改约束行为。
 
#### 指定主机组之外的主机执行
无论使用那种方式，我们都需要提供一份主机信息(ansible：hosts, salt：target)，实际中有一种场景，就是需要在执行任务模块中指定一个主机组之外的主机来执行一个动作。

这类的场景切入例如：搭建 mysql 主从时，需要在 master 主机上做对应的同步账号授权以及数据导出(不使用备份)，针对此种场景构建主机时只需要包含 slave 主机即可，然后对每个 slave 主机绑定上对应的 master 主机的信息即可，这样就会显得构建的主机信息层次非常清晰简练。

__Ansible 实现__
主要借助 ansible 的 delegate_to 特性来实现，以下为模拟搭建主从时同步导出数据，这里使用 nc 模拟
```shell
# cat hosts
[db]
192.168.1.81 master_host=192.168.1.80
 
# cat t1.yaml
- name: t1 playbook
  hosts: db
  remote_user: root
  gather_facts: True
  tasks:
    # 本机启动 nc 进程，准备接受 master 传输的数据
    - name: task 1
      shell: "nc -l 13306 | tar -zxf - -C /data/mysql/"
      async: 99999
      poll: 0

    # 确认 nc 后台进程是否正常监听启动
    - name: task 2
      shell: "ss -tnlp | grep nc | grep 13306"

    # 指定主机对应的 master 传输数据，注意此处的 master 为主机的变量而非定义的主机
    - name: task 3
      shell: "cd /data/mysql ; tar cfz - * | nc {{ inventory_hostname }} 13306"
      delegate_to: "{{ master_host }}"


# 确认 192.168.1.81 /data/mysql
# ll /data/mysql
total 0


# 执行 playbook
# ansible-playbook t1.yaml -i hosts
PLAY [t1 playbook] *********************************************************************************************************************************************
TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [192.168.1.81]
TASK [task 1] **************************************************************************************************************************************************
changed: [192.168.1.81]
TASK [task 2] **************************************************************************************************************************************************
changed: [192.168.1.81]
TASK [task 3] **************************************************************************************************************************************************
changed: [192.168.1.81 -> 192.168.1.80]
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.81               : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
 
# 确认 192.168.1.81 /data/mysql 文件
# ll /data/mysql
total 12
-rw-r--r-- 1 root root     0 May 10 17:15 a
drwxr-xr-x 2 root root     6 May 10 17:15 aa
-rw-r--r-- 1 root root     0 May 10 17:15 b
drwxr-xr-x 2 root root     6 May 10 17:15 bb
-rw-r--r-- 1 root root     0 May 10 17:15 c
drwxr-xr-x 2 root root     6 May 10 17:15 cc
-rw-r--r-- 1 root root     0 May 10 17:15 d
drwxr-xr-x 2 root root     6 May 10 17:15 dd
-rw-r--r-- 1 root root 10240 May 16 18:43 z
```
通过结果可以看出，192.168.1.81 把任务指派给了 192.168.1.80 而 192.168.1.80 未在主机组中定义，只要能保证 ansible 操作机的 ssh 权限就可以在整个操作过程中任意指派主机来执行定义的模块。

__Salt 实现__
主要借助 salt 的 event/reactor 特性来实现，这里同样使用 nc 模拟
```shell
# master 配置 reactor
# cat /etc/salt/master
...
reactor:
  - "general_reactor":
    - salt://reactors/cmd_exec.sls
...
 
# cat cmd_exec.sls
cmd_exec:
  local.cmd.shell:
    - tgt: {{ data['data']['host'] }}
    - arg:
        - "{{ data['data']['cmd_str'] }}"
 
# cat m6/t1.sls
m6_task_1:
  event.send:
    - name: general_reactor
    - data:
      host: {{ pillar[grains['id']]['master_host'] }}
      cmd_str: "sleep 5 ; cd /data/mysql ; tar cfz - * | nc {{ grains['id'] }} 13306"

m6_task_2:
  cmd.run:
    - name: "cd /data/mysql ; nc -l 13306 | tar zxf -"


# 192.168.1.81 主机确认文件状态
# ll /data/mysql
total 0
 
# salt master 执行 state
# salt "192.168.1.81" state.apply m6.t1 pillar='{"192.168.1.81": {"master_host": "192.168.1.80"}}'
192.168.1.81:
  Name: general_reactor - Function: event.send - Result: Changed Started: - 17:22:07.054790 Duration: 5.588 ms
  Name: cd /data/mysql ; nc -l 13306 | tar zxf - - Function: cmd.run - Result: Changed Started: - 17:22:07.062656 Duration: 5321.716 ms
Summary for 192.168.1.81
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   5.327 s


# 192.168.1.81 主机确认文件状态
# ll /data/mysql
total 12
-rw-r--r-- 1 root root     0 May 10 17:15 a
drwxr-xr-x 2 root root     6 May 10 17:15 aa
-rw-r--r-- 1 root root     0 May 10 17:15 b
drwxr-xr-x 2 root root     6 May 10 17:15 bb
-rw-r--r-- 1 root root     0 May 10 17:15 c
drwxr-xr-x 2 root root     6 May 10 17:15 cc
-rw-r--r-- 1 root root     0 May 10 17:15 d
drwxr-xr-x 2 root root     6 May 10 17:15 dd
-rw-r--r-- 1 root root 10240 May 16 18:43 z
```
由于 salt 与 ansible 本质上存在着运行机制的差别，所以不能像 ansible 那样直接指定(即 delegate_to)运行的主机，不过可以通过事件机制发送自定义事件至 salt 事件总线中，匹配自定义 reactor 即可完成 minion 端触发在执行 target 外的某台主机进行对应操作。

#### ansible、salt 主机组构建
如果需要临时固定一批主机来执行对应的操作(playbook 角色、state 公式)，通常需要给定工具一个执行的主机组，这里凭借经验来讲一般为主机 ip 构成的列表(因为不管是统配操作还是子组嵌套，针对的场景都较为固定，不够灵活)，所以此处以 ip 列表构成的主机组为背景。
 
注意：此处仅为介绍二者的差异，具体到每个工具的详细主机或 target 定义使用方式请见二者的官方文档。
 
此处分为两部分来阐述：

一、手动维护一个相对固定的主机组信息比如一个配置记录主机组信息的文件

__Ansible__
ansible 这部分的实现较为简单，可以修改默认的 hosts 文件，也可创建自定义的 hosts 文件
```shell
# cat hosts
[group1]
192.168.1.80
192.168.1.81
 
# ansible group1 -m ping -i hosts
192.168.1.80 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
192.168.1.81 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
__Salt__
salt 主机组的构建需要在 master 的配置文件中定义对应的 nodegroup，定义好后后续修改 nodegroup 的内容，不需要重启 salt-master
```shell
# cat /etc/salt/master
...
default_include: master.d/*.yaml
...
 
# 可以在自己的工程文件中定义好对应的 target 信息，然后创建软连即可
# mkdir /srv/nodegroups && ln -sv /srv/nodegroups /etc/salt/master.d/
# cd /srv/nodegroups && cat hosts.yaml
nodegroups: # 此处为固定配置关键字，不能更改
  group1:
    - 192.168.1.80
  group2:
    - 192.168.1.81
  group3:
    - 192.168.1.80
    - 192.168.1.81


# salt -N group3 test.ping
192.168.1.80:
    True
192.168.1.81:
    True
```
二、自定义的某种算法程序根据给定的参数动态生成主机组信息

一般在实际操作中常常碰到一些根据"逻辑信息"动态生成主机信息，例如根据给定的项目名返回整个项目的所有主机或是根据给定的 db 类型 master 返回所有类型为 master 的主机信息等，这样在结合 cmdb 这种架构集群拓扑生成的数据后就能够很顺畅的对接执行工具完成需要执行的操作。此处以自动生成 mysql 主从关系的主机组为例。
 
生成主从对应关系前提说明：给定三两台主机，根据一对二方式生成主机组，基于生成的主机组能够完成 db 的安装以及对应主从的搭建。

```
   +--------+
   | hostA  |
   +--------+                       
   
                       +--------+          +-------------+
   +--------+          | hostA  |  ----->  | hostC:3306  |
   | hostB  |  ====>   +--------+          +-------------+  
   +--------+          +--------+          +-------------+
                       | hostB  |  ----->  | hostC:3307  |
                       +--------+          +-------------+
   +--------+               
   | hostC  | 
   +--------+
```

__Ansible__
这里借助 ansible 的自定义模块以及 add_host 模块完成搭建主从主机数据结构的渲染。
```shell
# 目录结构，其中 library 为 ansible 自定义模块的目录（可根据实际需求配置调整）
# tree .
.
├── library
│   ├── generate_db_for_init.py
└── t3.yaml
1 directories, 2 files


# cat library/generate_db_for_init.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from ansible.module_utils.basic import *


def main():
    module = AnsibleModule(argument_spec=dict(
        port_numbers=dict(type='str', required=True,),
        newdb=dict(type='list', required=True),
    ))
    '''
    res_data = {
        "install": [
            { "ip": "10.155.162.71", "port_list": ["3306"]},
            { "ip": "10.173.166.135", "port_list": ["3306"]},
            { "ip": "10.155.114.12", "port_list": ["3306", "3307"]}
        ],
        "build_ms": []
    }
    '''
    PORT_NUMBERS = int(module.params["port_numbers"])
    NEWDB = module.params["newdb"]
    RES_DATA = {
        "install": [],
        "build_ms": [],
    }

    if PORT_NUMBERS == 1 and len(NEWDB) > 0:
        if len(NEWDB)%2 == 0:
            for i in NEWDB:
                tmp_install_info = {
                    "ip": i,
                    "port_list": ["3306"],
                }
                RES_DATA["install"].append(tmp_install_info)

            for j in range(0, len(NEWDB), 2):
                tmp_build_ms_info = {
                    "ip": NEWDB[j+1],
                    "db_info": {
                        "3306": NEWDB[j]
                    }
                }
                RES_DATA["build_ms"].append(tmp_build_ms_info)

    elif PORT_NUMBERS == 2 and len(NEWDB) > 0:
        if len(NEWDB)%3 == 0:
            for i in range(1, len(NEWDB)+1):
                if i%3 != 0:
                    tmp_install_info = {
                        "ip": NEWDB[i-1],
                        "port_list": ["3306"],
                    }
                elif i%3 == 0:
                    tmp_install_info = {
                        "ip": NEWDB[i-1],
                        "port_list": ["3306", "3307"]
                    }
                RES_DATA["install"].append(tmp_install_info)

            for j in range(0, len(NEWDB), 3):
                tmp_build_ms_info = {
                    "ip": NEWDB[j+2],
                    "db_info": {
                        "3306": NEWDB[j],
                        "3307": NEWDB[j+1],
                    }
                }

                RES_DATA["build_ms"].append(tmp_build_ms_info)

    if RES_DATA:
        module.exit_json(changed=True, failed=False, ansible_facts=dict(init_db_data=RES_DATA))
    else:
        module.fail_json(msg="Oh! no, Please check it on the target machine.")


if __name__ == '__main__':
    main()

# cat t3.yaml
- name: t3 playbook 1
  hosts: localhost
  remote_user: root
  gather_facts: False
  tasks:
    - name: task 1
      generate_db_for_init:
        port_numbers: 2
        newdb:
          - 192.168.1.80
          - 192.168.1.81
          - 192.168.1.82
    - name: task 2
      add_host:
        name: "{{ item.1['ip'] }}"
        groups: "build_db"
        sleep_time: "{{ (item.0)*5 }}"
        db_info: "{{ item.1['db_info'] }}"
      with_indexed_items: "{{ init_db_data['build_ms'] }}"

- name: t3 playbook 2
  hosts: "build_db"
  remote_user: root
  gather_facts: False
  tasks:
    - name: task 1
      debug:
        msg: "{{ inventory_hostname }} db_info: {{ db_info }}"
      connection: local


# ansible-playbook t3.yaml
PLAY [t3 playbook 1] *******************************************************************************************************************************************
TASK [task 1] **************************************************************************************************************************************************
changed: [localhost]
TASK [task 2] **************************************************************************************************************************************************
changed: [localhost] => (item=[0, {'ip': '192.168.1.82', 'db_info': {'3306': '192.168.1.80', '3307': '192.168.1.81'}}])
PLAY [t3 playbook 2] *******************************************************************************************************************************************
TASK [task 1] **************************************************************************************************************************************************
ok: [192.168.1.82] => {
    "msg": "192.168.1.82 db_info: {'3306': '192.168.1.80', '3307': '192.168.1.81'}"
}
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.82               : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
localhost                  : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
上述中第二个 playbook 中打印的数据 "msg": "192.168.1.82 db_info: {'3306': '192.168.1.80', '3307': '192.168.1.81'}" 即为动态生成的数据结构，实际上第二个 playbook 就是真正运行搭建主从的部分。构建的主机信息中 192.168.1.82 为真正的主机地址，对应的是 slave 的角色，而 192.168.1.82 主机变量 db_info 中 3306 对应的 master 为 192.168.1.80 以及 3307 对应的 master 的 192.168.1.81。

__Salt__
这里主要用到 salt 的 orchestrate 服务编排以及 py 的渲染器(可以内嵌执行 python 模块代码)。
```shell
# tree .
.
├── m8
│   ├── init.sls
│   └── t1.sls
├── orch
│   ├── o5.sls
│   └── plugins
│       ├── generate_db_for_init.py

3 directories, 3 files
 
# cat m8.t1.sls
m8_task_1:
  cmd.run:
    - name: "echo {{ pillar[grains['id']] }}"


# cat orch.o5.sls
#!py
import sys
sys.path.append('/root/workdir/ops/saltstack/salt/orch')


from plugins.generate_db_for_init import generate_data

def run():
    """
    generate data for build db
    """
    port_number = __pillar__.get('port_number', 0)
    newdb = __pillar__.get('newdb', [])
    res_data = generate_data(port_number, newdb)
    final_data = {}
    target_list = []
    for item in res_data['build_ms']:
        target_list.append(item['ip'])
        final_data[item['ip']] = {
            "db_info": item['db_info']
        }
    return {
        "o5_t1": {
            "salt.function": [
                 {"name": "state.apply"},
                 {"tgt": target_list},
                 {"tgt_type": "list"},
                 {"arg": ["m8"]},
                 {"kwarg": {"pillar": final_data}}
            ]
        },
    }


# cat orch/plugins/generate_db_for_init.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

def generate_data(port_numbers, newdb):
    RES_DATA = {
        "install": [],
        "build_ms": [],
    }
    PORT_NUMBERS = port_numbers
    NEWDB = newdb
    if PORT_NUMBERS == 1 and len(NEWDB) > 0:
        if len(NEWDB)%2 == 0:
            for i in NEWDB:
                tmp_install_info = {
                    "ip": i,
                    "port_list": ["3306"],
                }
                RES_DATA["install"].append(tmp_install_info)
            for j in range(0, len(NEWDB), 2):
                tmp_build_ms_info = {
                    "ip": NEWDB[j+1],
                    "db_info": {
                        "3306": NEWDB[j]
                    }
                }
                RES_DATA["build_ms"].append(tmp_build_ms_info)
    elif PORT_NUMBERS == 2 and len(NEWDB) > 0:
        if len(NEWDB)%3 == 0:
            for i in range(1, len(NEWDB)+1):
                if i%3 != 0:
                    tmp_install_info = {
                        "ip": NEWDB[i-1],
                        "port_list": ["3306"],
                    }
                elif i%3 == 0:
                    tmp_install_info = {
                        "ip": NEWDB[i-1],
                        "port_list": ["3306", "3307"]
                    }
                RES_DATA["install"].append(tmp_install_info)
            for j in range(0, len(NEWDB), 3):
                tmp_build_ms_info = {
                    "ip": NEWDB[j+2],
                    "db_info": {
                        "3306": NEWDB[j],
                        "3307": NEWDB[j+1],
                    }
                }
                RES_DATA["build_ms"].append(tmp_build_ms_info)
    if RES_DATA:
        return RES_DATA
    else:
        return {}

if __name__ == '__main__':
    main()


# salt-run --state-output full state.orchestrate orch.o5 pillar='{"port_number": 2, "newdb": ["192.168.1.80","192.168.1.81", "192.168.1.99"]}'
tangcw-dev_master:
----------
          ID: o5_t1
    Function: salt.function
        Name: state.apply
      Result: True
     Comment: Function ran successfully. Function state.apply ran on 192.168.1.99.
     Started: 08:45:06.594327
    Duration: 292.058 ms
     Changes:
              192.168.1.99:
              ----------
                        ID: m8_task_1
                  Function: cmd.run
                      Name: echo {'db_info': {'3306': '192.168.1.80', '3307': '192.168.1.81'}}
                    Result: True
                   Comment: Command "echo {'db_info': {'3306': '192.168.1.80', '3307': '192.168.1.81'}}" run
                   Started: 08:45:06.870835
                  Duration: 7.785 ms
                   Changes:
                            ----------
                            pid:
                                17711
                            retcode:
                                0
                            stderr:
                            stdout:
                                {db_info: {3306: 192.168.1.80, 3307: 192.168.1.81}}
              Summary for 192.168.1.99
              ------------
              Succeeded: 1 (changed=1)
              Failed:    0
              ------------
              Total states run:     1
              Total run time:   7.785 ms
Summary for tangcw-dev_master
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time: 292.058 ms
```
可见 stdout 输出内容与之前 ansible 一致，此处因为 salt 的 pillar 变量的特性，所以借助 grains 的 id 获取对应每个 target 对应的主机变量。

__总结__
基于不同的业务场景构建一个健全的主机数据结构是很有必要的，而通过动态的特性只需要给与特定参数即可按需生成对应的主机数据结构进而完成对应的业务操作。也能无缝的和 "cmdb" 等平台完成对接(此处后续会详细说明)。上述在执行过程中真正需要给定的参数为 port_numbers、newdb，ansible 可以通过命令行 -e 'vars' 或者通过 -e '@file.yaml' 传入，而 salt 则通过 pillar='vars' 选项或者 pillar=$(generate_vars_scripts) 输入。
 
#### 限制指定部分任务
不管是 ansible 的 roles 还是 salt 的公式，通常都是需要执行的一系列任务的封装集合，而实际常常有种需求为单次调用任务集时恰恰只需执行其中部分任务，当然这种情况需要在执行的时候给定一些参数来实现。
 
__Ansible__
ansible 可以通过 tag 标签属性来实现。
```shell
# cat t4.yaml
- name: t1 playbook
  hosts: 192.168.1.80
  remote_user: root
  gather_facts: False
  tasks:
    - name: task 1
      debug:
        msg: "task 1"
      tags:
        - tag1
        - tag3
    - name: task 2
      debug:
        msg: "task 2"
      tags:
        - tag2
        - tag3
 
# 默认不给定 tag，执行所有
# ansible-playbook t4.yaml
PLAY [t1 playbook] *********************************************************************************************************************************************
TASK [task 1] **************************************************************************************************************************************************
ok: [192.168.1.80] => {
    "msg": "task 1"
}
TASK [task 2] **************************************************************************************************************************************************
ok: [192.168.1.80] => {
    "msg": "task 2"
}
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.80               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


# 只执行 tag：tag1 的任务
# ansible-playbook t4.yaml -t tag1
PLAY [t1 playbook] *********************************************************************************************************************************************
TASK [task 1] **************************************************************************************************************************************************
ok: [192.168.1.80] => {
    "msg": "task 1"
}
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.80               : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
 
# 执行除 tag1 之外所有的任务
# ansible-playbook t4.yaml --skip-tags tag1
PLAY [t1 playbook] *********************************************************************************************************************************************
TASK [task 2] **************************************************************************************************************************************************
ok: [192.168.1.80] => {
    "msg": "task 2"
}
PLAY RECAP *****************************************************************************************************************************************************
192.168.1.80               : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
ansible 中除了 task 外还可以对 play、role、include打一个tag(标签)来限制执行，此处详见 [ansible_tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html)
 
__Salt__
salt state 本身支持选择 sls 任务模块列表如：salt '*' state.apply stuff,pkgs，指定 exclude="[{'id': 'id_to_exclude'}, {'sls': 'sls_to_exclude'}]" 反向排除不需要执行的任务，亦或是选择需要执行的单个 task_id 如：salt '*' state.sls_id my_state my_module pillar='{"foo": "bar"}', 但总体上而言不如 ansible 的 tag 那样更为灵活，不过我们可以通过 salt 的 jinja render 来达到类似的实现。
```shell
# 自定义模块：list_tag_filter
# cd _modules && cat check_tag.py
# -*- coding: utf-8 -*-
def list_tag_filter(tags):
    current_tags = __pillar__.get('tags', ['all'])
    if 'all' not in tags:
        tags.append('all')
    for current_tag in current_tags:
        if current_tag in tags:
            return True
    return False


# salt "*" saltutil.sync_modules
 
# cat m9/t1.sls
{% if salt['check_tag.list_tag_filter'](['tag1']) %}
m9_t1_task_1:
  cmd.run:
    - name: "echo 'x'"
{% endif %}

{% if salt['check_tag.list_tag_filter'](['tag2']) %}
m9_t1_task_2:
  cmd.run:
    - name: "echo 'o'"
{% endif %}
 
# 执行 tag1 的任务
# salt -N group1 state.apply m9.t1 pillar='{"tags": ["tag1"]}'
192.168.1.80:
  Name: echo 'x' - Function: cmd.run - Result: Changed Started: - 15:48:50.607122 Duration: 10.303 ms
Summary for 192.168.1.80
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:  10.303 ms


# 执行 tag2 的任务
# salt -N group1 state.apply m9.t1 pillar='{"tags": ["tag2"]}'
192.168.1.80:
  Name: echo 'o' - Function: cmd.run - Result: Changed Started: - 15:49:18.202508 Duration: 11.264 ms
Summary for 192.168.1.80
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:  11.264 ms


# 不给定默认执行所有
# salt -N group1 state.apply m9.t1 pillar='{}'
192.168.1.80:
  Name: echo 'x' - Function: cmd.run - Result: Changed Started: - 15:49:39.174939 Duration: 11.209 ms
  Name: echo 'o' - Function: cmd.run - Result: Changed Started: - 15:49:39.186559 Duration: 8.631 ms
Summary for 192.168.1.80
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:  19.840 ms
```
结合 salt 的 pillar 与 jinja 的渲染机能能够完成更为灵活的条件匹配进而达到限制执行的目录。
 
__总结__
在编写角色或者公式中不管 ansible 还是 salt 限制执行都不是必须的，不过在实际场景中为了能够让执行集合适配更多的执行调用场景，除了更细化的文件模块拆分，对相关任务做划分类别的标签处理能大大提供封装工具的通用性。
 
#### 任务进度一致性
TODO
#### 并行、异步任务
TODO 
#### ansible playbook 角色/salt state 公式 的变量构建
TODO 
## 项目工程化
TODO
## Ansbile 与 saltstack 操作 UI
TODO
 
## 接入运维平台
TODO
 
## 接入公有云平台
TODO
