# Ansible Monitoring Project

✨ **Technologies**: Ansible, Prometheus, Prometheus Node Exporter, Grafana

**Summary**: Using Ansible, install Prometheus Node Exporter on all on-premises Raspberry Pis, and set up Prometheus and Grafana on a central Raspberry Pi to collect metrics from the Node Exporters of the other Raspberry Pis, creating a monitoring solution.

**Project folder structure**
```bash
.
├── README.md
├── inventory
├── playbook.yaml
└── roles
    ├── grafana
    │   ├── README.md
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   │   └── grafana.conf.j2
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    ├── prometheus
    │   ├── README.md
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   │   ├── init.service.j2
    │   │   └── prometheus.yml.j2
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    └── prometheus_node_exporter
        ├── README.md
        ├── defaults
        │   └── main.yml
        ├── files
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        ├── templates
        │   └── init.service.j2
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml
```


Credit to Devops-Techstack
https://youtu.be/3id6l_BWdNA?si=oBixlhHEbjDwXEoq
https://github.com/Devops-Techstack/Ansible-RealTime-Projects

## 1) Initializing Project

```bash
mkdir ansible-monitoring-project
cd ./ansible-monitoring-project

touch inventory
touch playbook.yml

mkdir roles
cd roles

ansible-galaxy init prometheus_node_exporter
ansible-galaxy init prometheus
ansible-galaxy init grafana
```

**inventory**
```ini
[central_server]
localhost ansible_connection=local

[all_servers]
localhost ansible_connection=local
192.168.50.120
192.168.50.67
192.168.50.84
```

**playbook.yml**
```yaml
---
- hosts: all_servers
  become: yes
  roles:
    - prometheus_node_exporter
- hosts: central_server
  become: yes
  roles:
    - grafana
    - prometheus
```

## 2) Set up connection to servers
On the central server, create a key pair.
```bash
ssh-keygen -t rsa       
```
Copy the public key content and paste to servers' authorized_keys.
```bash
vi ~/.ssh/authorized_keys
# paste the public key content
```

## 3) Set up Prometheus Node Exporter Role
**templates/init.service.j2**
```ini
[Unit]
Description={{serviceName}}
Wants=network-online.target
After=network-online.target

[Service]
User={{ userId }}
Group={{ groupId }}
Restart=on-failure
Type=simple
ExecStart={{ exec_command }}

[Install]
WantedBy=multi-user.target
```

**vars/main.yml**
```yaml
---
serviceName: "node_exporter"
userId: "node_exporter"
groupId: "node_exporter"
exec_command: /usr/local/bin/node_exporter
version: 0.16.0
```

**tasks/main.yml**
```yaml
---
# tasks file for prometheus_node_exporter
- name: Creating node_exporter user group
  group: name="{{groupId}}"
  become: true

- name: Creating node_exporter user
  user:
    name: "{{userId}}"
    group: "{{groupId}}"
    system: yes
    shell: "/sbin/nologin"
    comment: "{{userId}} nologin User"
    createhome: "no"
    state: present

- name: Install prometheus node exporter
  unarchive:
    src: "https://github.com/prometheus/node_exporter/releases/download/v{{ version }}/node_exporter-{{ version }}.linux-armv7.tar.gz"
    dest: /tmp/
    remote_src: yes

- name: Copy prometheus node exporter file to bin
  copy:
    src: "/tmp/node_exporter-{{ version }}.linux-armv7/node_exporter"
    dest: "/usr/local/bin/node_exporter"
    owner: "{{userId}}"
    group: "{{groupId}}"
    remote_src: yes
    mode: 0755

- name: Delete node exporter tmp folder
  file:
    path: '/tmp/node_exporter-{{ version }}.linux-armv7'
    state: absent

- name: Copy systemd init file
  template:
    src: init.service.j2
    dest: /etc/systemd/system/node_exporter.service

- name: Start node_exporter service
  service:
    name: node_exporter
    state: started
    enabled: yes

- name: Check if node exporter emits metrices
  uri:
    url: http://127.0.0.1:9100/metrics
    method: GET
    status_code: 200
```

* Change accordingly
Raspberry Pi: linux-armv7
AWS EC2: linux-amd64

## 4) Set up Prometheus Role
**templates/init.service.j2**
```ini
[Unit]
Description={{serviceName}}
Wants=network-online.target
After=network-online.target

[Service]
User={{ userId }}
Group={{ groupId }}
Restart=on-failure
Type=simple
ExecStart={{ exec_command }}

[Install]
WantedBy=multi-user.target
```

**templates/prometheus.yml.j2**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets:
        {% for host in groups['all_servers'] %}
        {% if inventory_hostname != host %}
        - '{{ host }}:9100'
        {% endif %}
        {% endfor %}
```
**❗ Warning**: There could be an indentation problem for prometheus.yml.j2 after running Ansible. If there is an error, check /etc/prometheus/prometheus.yml.

**vars/main.yml**
```yaml
---
serviceName: "prometheus"
userId: "prometheus"
groupId: "prometheus"
exec_command: "/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.conf --storage.tsdb.path=/data/prometheus --storage.tsdb.retention=2d"
version: "2.3.2"
```

**tasks/main.yml**
```yaml
---
# tasks file for prometheus
- name: Creating prometheus user group
  group: name="{{groupId}}"
  become: true

- name: Creating prometheus user
  user:
    name: "{{userId}}"
    group: "{{groupId}}"
    system: yes
    shell: "/sbin/nologin"
    comment: "{{userId}} nologin User"
    createhome: "no"
    state: present

- name: Install prometheus
  unarchive:
    src: "https://github.com/prometheus/prometheus/releases/download/v{{ version }}/prometheus-{{ version }}.linux-armv7.tar.gz"
    dest: /tmp/
    remote_src: yes

- name: Copy prometheus file to bin
  copy:
    src: "/tmp/prometheus-{{ version }}.linux-armv7/prometheus"
    dest: "/usr/local/bin/prometheus"
    owner: "{{userId}}"
    group: "{{groupId}}"
    remote_src: yes
    mode: 0755

- name: Delete prometheus tmp folder
  file:
    path: '/tmp/prometheus-{{ version }}.linux-armv7'
    state: absent

- name: Creates directory
  file: 
    path: "/data/prometheus/"
    state: directory
    owner: "{{userId}}"
    group: "{{groupId}}"
    mode: 0755

- name: Creates directory
  file: 
    path: "/etc/prometheus/"
    state: directory
    owner: "{{userId}}"
    group: "{{groupId}}"
    mode: 0755

- name: config file
  template:
    src: prometheus.yml.j2
    dest: /etc/prometheus/prometheus.yml

- name: Copy systemd init file
  template:
    src: init.service.j2
    dest: /etc/systemd/system/prometheus.service
  notify: systemd_reload

- name: Start prometheus service
  service:
    name: prometheus
    state: started
    enabled: yes

- name: Check if prometheus is accessible
  uri:
    url: http://localhost:9090
    method: GET
    status_code: 200
```

* Change accordingly
Raspberry Pi: linux-armv7
AWS EC2: linux-amd64

**handlers/main.yml**
``` yaml
- name: Restart the Prometheus service
  service:
    name: prometheus
    state: restarted
  listen: event_restart_prometheus

- name: Reload systemd
  command: systemctl daemon-reload
  listen: systemd_reload
```

## 5) Set up Grafana Role
**templates/grafana.conf.j2**
Copy the content from the link and paste to the file.
Link: https://github.com/grafana/grafana/blob/main/conf/sample.ini

**vars/main.yml**
```bash
---
version: 5.4.3
```

**tasks/main.yml**
``` yaml
---
# tasks file for grafana
- name: Create APT keyring directory for Grafana
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Download and add Grafana GPG key
  ansible.builtin.apt_key:
    url: https://apt.grafana.com/gpg.key
    keyring: /etc/apt/keyrings/grafana.gpg

- name: Add Grafana repository
  ansible.builtin.apt_repository:
    repo: 'deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main'
    state: present
    filename: grafana

- name: Install Grafana
  ansible.builtin.apt:
    name: grafana
    state: latest
    update_cache: yes

- name: Ensure Grafana is running and enabled
  ansible.builtin.systemd:
    name: grafana-server
    enabled: yes
    state: started
```

**handlers/main.yml**
``` yaml
---
# handlers file for grafana
- name: "Restart the Grafana service."
  service:
    name: grafana-server
    state: restarted
  listen: event_restart_grafana
```

## 6) Run Ansible Playbook
```bash
ansible-playbook -i inventory_file playbook.yml -K [become password]
```



## * Troubleshooting

```bash
# Check log
ststemctl status prometheus
journalctl -u prometheus

# Start the Prometheus server manually to troubleshoot the Prometheus server or apply test configurations.
sudo /usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus --storage.tsdb.retention=2d

# Make changes and restart prometheus
sudo systemctl reload daemon
sudo systemctl restart prometheus
```
