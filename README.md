# ElasticSearchForEventService_testing



refer to Wiki for detailed setup instructions

![](https://github.com/wouyang628/ElasticSearchForEventService_testing/blob/master/images/flow.jpg)

# Elasticseach
Following https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html#deb
```
root@ubuntu:~# wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
root@ubuntu:~# apt-get install apt-transport-https
root@ubuntu:~# echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
root@ubuntu:~# apt-get update && sudo apt-get install elasticsearch
root@ubuntu:~# /bin/systemctl daemon-reload
root@ubuntu:~# /bin/systemctl enable elasticsearch.service
root@ubuntu:~# systemctl start elasticsearch.service
root@ubuntu:~# curl -X GET "localhost:9200/?pretty"
{
  "name" : "ubuntu",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "lEMLDDdLTz6IH0v9SC747Q",
  "version" : {
    "number" : "7.4.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "fc0eeb6e2c25915d63d871d344e3d0b45ea0ea1e",
    "build_date" : "2019-10-22T17:16:35.176724Z",
    "build_snapshot" : false,
    "lucene_version" : "8.2.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

Install the elasticsearch package with pip:
```
root@ubuntu:~/monitoring# pip3 install elasticsearch
```
to add a new event:
```
            new_message = {
                "device_ID": self.get_device_ID(),
                "device_type": "network",
                "timestamp": timeNow,
                "vendor": "Juniper",
                "element_name" : self.get_element_name(),
                "error_code": self.get_error_code(),
                "error_message": self.get_message(),
                "status": "new",
                "action": "none",
                "source": "healthbot",

import requests
import json
import datetime
from elasticsearch import Elasticsearch

timeNow = datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")
event1 = {
    "timestamp": timeNow,
    "device_ID": "vmx101", 
    "device_type": "network",
    "vendor": "Juniper", 
    "element_name": "ge-0/0/0", 
    "error_code": "interface_down", 
    "error_message": "ge-0/0/0 is down", 
    "status": "new", 
    "action": "none",
    "source": "healthbot",
    "result": "none"
}

event2 = {
    "timestamp": timeNow,
    "device_ID": "linux2", 
    "device_type": "server",
    "vendor": "unknow", 
    "element_name": "cpu", 
    "error_code": "cpu_100", 
    "error_message": "cpu 100", 
    "status": "new", 
    "action": "none",
    "source": "healthbot",
    "result": "none"
}
es = Elasticsearch([{'host': 'localhost', 'port': 9200}])
es.index(index='events', body = event1)
es.index(index='events', body = event2)
```
to query for new events:
```
es_result = es.search(index="events", body={"query": {"match": {'status':'new'}}})
```
to get all document:
```
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_all": {}
    }
}
'
```
```
es_result = es.search(index="events", body={"query": {"match_all": {}}})
for event in es_result["hits"]["hits"]:
    print(event)

{'_id': 'QnNx-m0BWRj0BTxBNHR7', '_index': 'events', '_source': {'device_type': 'network', 'vendor': 'juniper', 'element_name': 'ge-0/0/0', 'timestamp': '2019-10-23T14:05:10.670438', 'error_code': 'interface_down', 'device_ID': 'vmx101', 'error_message': 'ge-0/0/0 is down', 'status': 'processed', 'result': 'new'}, '_type': '_doc', '_score': 1.0}
{'_id': 'Q3PX-m0BWRj0BTxBUXSM', '_index': 'events', '_source': {'device_type': 'server', 'vendor': 'unknow', 'element_name': 'cpu', 'timestamp': '2019-10-23T15:57:49.581363', 'error_code': 'cpu_100', 'device_ID': 'linux2', 'error_message': 'cpu 100', 'status': 'new', 'result': 'none'}, '_type': '_doc', '_score': 1.0}
```
to find the document id:
```
print(es_result["hits"]["hits"][0]["_id"])
```
to update an event:
```
source_to_update = {
    "doc": {
        "status": "processed"
    }
}
response = es.update(index="events", id="QnNx-m0BWRj0BTxBNHR7", body=source_to_update)
```


# Kibana
# Setting Up Kibana

following this guide for Kibana install and setup https://www.elastic.co/guide/en/kibana/current/deb.html
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
apt-get update && sudo apt-get install kibana
/bin/systemctl daemon-reload
/bin/systemctl enable kibana.service
systemctl start kibana.service
```

```
vi /etc/kibana/kibana.yml

server.host: "<ip address of the Kibana server>"
```

# SMTP

To enable sending email , please configure SNMP following https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-16-04

e.g.
```
import smtplib
sender = 'event_service'
receivers = ['']
message = """From: From Event Service
To: To You
Subject: Event Cannot be handled

Event Cannot be handled.
Error Information:
{}
""".format(event)
smtpObj = smtplib.SMTP('localhost')
smtpObj.sendmail(sender, receivers, message)
```

# Stage the linux server for reboot  
create a sudo user on linux2 . 
```
[root@linux2 ~]# adduser mememe
[root@linux2 ~]# passwd mememe
Changing password for user mememe.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[root@linux2 ~]# usermod -aG wheel mememe
```
setting up keyless remote login to the remote linux2 server from the server running the remediation program(ubuntu3)
```
[mememe@linux2 ~]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/mememe/.ssh/id_rsa):
Created directory '/home/mememe/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/mememe/.ssh/id_rsa.
Your public key has been saved in /home/mememe/.ssh/id_rsa.pub.
The key fingerprint is:
34:c8:5a:e4:f1:05:65:8c:b1:c6:88:7c:10:a6:d9:64 mememe@linux2
The key's randomart image is:
+--[ RSA 2048]----+
|    E.o o*+      |
|   O * *.+.      |
|  o + B B        |
|     + o .       |
|    .   S        |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```
add ubuntu3's public key to linux2's authorized keys:
```
root@ubuntu:~# cat .ssh/id_rsa.pub | ssh mememe@10.49.66.252 'cat >> .ssh/authorized_keys'
```
change permission:
```
root@ubuntu:~# ssh mememe@10.49.66.252 "chmod 700 .ssh; chmod 640 .ssh/authorized_keys"
```
verify that you can login from ubuntu3 to linux2 without password:
```
root@ubuntu:~# ssh mememe@10.49.66.252
Last login: Wed Oct 23 13:58:04 2019 from 10.49.64.214
```
on linux2 server, add the following in /etc/sudoers file so that the user mememe can execute the sudo command without type password
```
mememe ALL=(ALL) NOPASSWD:ALL
```

