# salt returner

| �汾| ���� | ״̬ |   �޶���  |    ժҪ  |
| ------ | ----- | ----- | ------- | ------ |
| v1.1  | 2017-07-10  | ���� |  ��Դ | salt |


> �ٷ���ַhttps://docs.saltstack.com/en/latest/ref/returners/

��minion��ִ�к�Ľ������һ���Ƿ��ص�master�˵ģ�������þ�����������ִ�еĽ�����ݴ洢��
�ط����������ļ����ڴ桢���ݿ⡢��Ϣ���еȵȣ���Ҫ�Ϳ����ȥдreturner��

# ���𻷾�

| ����   |   ��ɫ   |   ����ϵͳ |   ����汾  |    ��ע  |
| ------ | ----- | ----- | ------- | ------ |
| hz01-online-ops-salt-01(172.16.8.11)  | salt-master  |   Centos 7.3(x86-64)|  salt-master,salt-api |  ���ڵ�|
| hz01-online-ops-salt-02(172.16.8.12)  | salt-master  |   Centos 7.3(x86-64)|  salt-master,salt-api |  ���ڵ�|
| hz01-online-ops-salt-03(172.16.8.13)  | salt-minion |   Centos 7.3(x86-64)|  salt-minion |  slave�ڵ� |


- �ø���򵥵������罫���д���ļ�

> �Զ���returnerĬ��·����/srv/salt/_returners

```
[master]# mkdir -p /srv/salt/_returners
```

- �������returner

```
[master]# cat file.py 
import json
import time
 
def returner(ret):
    now = time.localtime() 
    now = time.strftime("%Y-%m-%d %H:%M:%S",now)
    result_file = '/tmp/returner'
    result = file(result_file,'a+')
    result.write('At'+str(json.dumps(now))+'\n')
    result.write(str(json.dumps(ret))+'\n')
    result.close()
```

- ͬ��returners

```
[master]# salt '*' saltutil.sync_returners 
hz01-online-ops-salt-01:
    - returners.file
hz01-online-ops-salt-02:
    - returners.file
hz01-online-ops-salt-03:
    - returners.file
```

- ִ������

```
[master]# salt '*' cmd.run 'hostname' --return file
```

- �鿴returner�洢����־�ļ�

```
# salt '*' cmd.run 'cat /tmp/returner'
hz01-online-ops-salt-01:
    At"2018-06-06 17:11:09"
    {"fun_args": ["hostname"], "jid": "20180606171109870043", "return": "hz01-online-ops-salt-01", "retcode": 0, "success": true, "fun": "cmd.run", "id": "hz01-online-ops-opennode-01.cs1cloud.internal"}
hz01-online-ops-salt-02:
    At"2018-06-06 17:11:09"
    {"fun_args": ["hostname"], "jid": "20180606171109870043", "return": "hz01-online-ops-salt-02", "retcode": 0, "success": true, "fun": "cmd.run", "id": "hz01-online-ops-opennode-02.cs1cloud.internal"}
hz01-online-ops-salt-03:
    At"2018-06-06 17:11:09"
    {"fun_args": ["hostname"], "jid": "20180606171109870043", "return": "hz01-online-ops-salt-03", "retcode": 0, "success": true, "fun": "cmd.run", "id": "hz01-online-ops-opennode-03.cs1cloud.internal"}
```






















