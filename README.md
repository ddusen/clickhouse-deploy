# Ansible Playbook for ClickHouse Cluster

### 第 1 步：在中控机上安装系统依赖包
```bash
yum -y install epel-release git curl sshpass && \
yum -y install python3-pip
```

### 第 2 步：在中控机上创建 clickhouse 用户，并生成 SSH key
```bash
#1.创建 clickhouse 用户
useradd -m -d /home/clickhouse clickhouse
#2.设置 clickhouse 用户密码(iKp****tM4d)
passwd clickhouse
#3.配置 clickhouse 用户 sudo 免密码，将 clickhouse ALL=(ALL) NOPASSWD: ALL 添加到文件末尾即可
echo 'clickhouse ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
#4.生成 SSH key
su - clickhouse
ssh-keygen -t rsa
```

### 第 3 步：在中控机器上下载 ClickHouse Ansible
```bash
# gitee(二选一)
git clone https://gitee.com/dusenn/clickhouse-ansible.git

# github(二选一)
git clone https://github.com/dusennn/clickhouse-ansible.git
```

### 第 4 步：在中控机器上安装 ClickHouse Ansible 及其依赖
```bash
cd /home/clickhouse/clickhouse-ansible && \
    python3 -m venv venv && \
    source venv/bin/activate && \
    pip install -i https://pypi.mirrors.ustc.edu.cn/simple/ --upgrade pip && \
    pip install -i https://pypi.mirrors.ustc.edu.cn/simple/ -r ./requirements.txt
```

### 第 5 步：在中控机上配置部署机器 SSH 互信及 sudo 规则
- 将你的部署目标机器 IP 添加到 hosts.ini 文件的 [servers] 区块下
```bash
cd /home/clickhouse/clickhouse-ansible && \
vi hosts.ini
```

- Examples:
```ini
[servers]
172.30.7.189
172.30.7.190
172.30.7.191

[all:vars]
username = clickhouse
ntp_server = pool.ntp.org
```

- 执行以下命令，按提示输入部署目标机器的 root 用户密码(iKp****tM4d)
```bash
ansible-playbook -i hosts.ini create_users.yml -u root -k 
```

### 第 6 步：挂载磁盘
```bash
#1.查看数据盘
ansible -i hosts.ini all -m shell -a "fdisk -l" -u clickhouse -b
#2.创建分区表
ansible -i hosts.ini all -m shell -a "parted -s -a optimal /dev/vdb mklabel gpt -- mkpart primary ext4 1 -1" -u clickhouse -b
#3.格式化文件系统
ansible -i hosts.ini all -m shell -a "mkfs.ext4 /dev/vdb1" -u clickhouse -b
#4.手动挂载
ansible -i hosts.ini all -m shell -a "mkdir /data1 && mount /dev/vdb1 /data1" -u clickhouse -b
#5.设置开启自动挂载
ansible -i hosts.ini all -m shell -a 'echo "/dev/vdb1 /data1 ext4 defaults 0 0" >> /etc/fstab' -u clickhouse -b
```

### 第 7 步：编辑 inventory.ini 文件，分配机器资源

> 集群架构:

- Sharding

|HostName|IP|实例1端口|实例1磁盘|实例2端口|实例2磁盘|
|---|---|---|---|---|---|
|clickhouse.sharding1.db|172.30.7.189|8123,8000,8004,8009|/data1 512GB|9123,9000,9004,9009|/data2 512GB|
|clickhouse.sharding2.db|172.30.7.190|8123,8000,8004,8009|/data1 512GB|9123,9000,9004,9009|/data2 512GB|
|clickhouse.sharding3.db|172.30.7.191|8123,8000,8004,8009|/data1 512GB|9123,9000,9004,9009|/data2 512GB|

- Sharding<->Replica

||replica1|replica2|
|---|---|---|
|sharding1|172.30.7.189|172.30.7.190|
|sharding2|172.30.7.190|172.30.7.191|
|sharding3|172.30.7.191|172.30.7.189|


```
## clickhouse Cluster Part
[clickhouse_servers]
172.30.7.189
172.30.7.190
172.30.7.191

[clickhouse_replica_servers]
172.30.7.189
172.30.7.190
172.30.7.191

[clickhouse_clients]
172.30.7.189

## Monitoring Part
# prometheus and pushgateway servers
[monitoring_servers]

[grafana_servers]

# node_exporter and blackbox_exporter servers
[monitored_servers]

## Global variables
[all:vars]
## Connection
# ssh via normal user
ansible_user = clickhouse

cluster_name = clickhouse-cluster

# CPU architecture: amd64, arm64
cpu_architecture = amd64

clickhouse_version = 20.8.7.15

# process supervision, [systemd, supervise]
process_supervision = systemd

timezone = Asia/Shanghai

enable_firewalld = False
# check NTP service
enable_ntpd = True
set_hostname = False
```

### 第 8 步：部署 ClickHouse 集群

> ansible-playbook 执行 Playbook 时，默认并发为 5。部署目标机器较多时，可添加 -f 参数指定并发数，例如 ansible-playbook deploy.yml -f 10。以下示例使用 clickhouse 用户作为服务运行用户：

#### 1. 在 clickhouse-ansible/inventory.ini 文件中，确认 ansible_user = clickhouse
```
## Connection
# ssh via normal user
ansible_user = clickhouse
```
> 注意：
> 不要将 ansible_user 设置为 root 用户，因为 clickhouse-ansible 限制了服务以普通用户运行。

- 执行以下命令，如果所有 server 均返回 clickhouse，表示 SSH 互信配置成功：
```bash
ansible -i inventory.ini clickhouse* -m shell -a 'whoami'
```

- 执行以下命令，如果所有 server 均返回 root，表示 clickhouse 用户 sudo 免密码配置成功。
```bash
ansible -i inventory.ini clickhouse* -m shell -a 'whoami' -b
```

#### 2. 执行 local_prepare.yml playbook，联网下载 ClickHouse RPM 至中控机。

```bash
ansible-playbook local_prepare.yml
```

#### 3. 初始化系统环境，修改内核参数。

```bash
ansible-playbook bootstrap.yml
```

#### 4. 部署 ClickHouse 集群软件。

```bash
ansible-playbook deploy.yml
```

#### 5. 启动 ClickHouse 集群。

```bash
ansible-playbook start.yml
```
#### 6. 其它命令
```bash
# 滚动更新 ClickHouse 集群
ansible-playbook rolling_update.yml


# 彻底清除 ClickHouse 集群
ansible-playbook unsafe_cleanup.yml
```


*****

Refs:

- https://github.com/pingcap/tidb-ansible
