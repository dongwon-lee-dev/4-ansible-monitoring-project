# Corporate Level DevOps Pipeline Project

✨ **Technologies**: Ansible, Prometheus, Prometheus Node Exporter, Grafana

Inspired by jaiswaladi246
https://youtu.be/NnkUGzaqqOc?si=-5ugADFn6lgBzpK9

# <span style="background-color: cyan;">1) Setting up Hosts</span>
### 1. Create VPC
project/
├── inventory
├── playbook.yml
└── roles/
    ├── prometheus_node_exporter/
    ├── prometheus/
    └── grafana/

### 2. Create Security Group 
```ini
# inventory
[monitor_server]
18.219.214.78

[all_servers]
18.219.214.78
18.222.105.189
```
*** admin인지 root인지 하나만 정해서 확실히 하나만 하기
ssh-keygen -t rsa       
~/.ssh/authorized_keys

ansible-playbook -i inventory_file playbook.yml -k private_key

### 3. Create EC2 Instances
```yaml
# playbook.yml
---
- hosts: all_servers
  become: yes
  roles:
    - prometheus_node_exporter

- hosts: monitor_server
  become: yes
  roles:
    - prometheus

- hosts: monitor_server
  become: yes
  roles:
    - grafana
```

```bash
ansible-galaxy init prometheus_node_exporter
ansible-galaxy init prometheus
ansible-galaxy init grafana
```

```bash
ansible-playbook -i inventory playbook.yaml
```

```bash
linux-amd64

# Raspberry Pi 
linux-armv7
```

# <span style="background-color: cyan;">2) Setting up Ansible</span>

Grafana config template
https://github.com/grafana/grafana/blob/main/conf/sample.ini

sudo systemctl reload daemon
sudo systemctl restart prometheus

journalctl -u prometheus

sudo /usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus --storage.tsdb.retention=2d
