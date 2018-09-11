# Prepare_ELK_Server_with_Debian_9
## Software Installation

```
apt update && apt install -y vim mc uml-utilities ntp qemu-guest-agent \
htop sudo curl git-core etckeeper molly-guard apt-transport-https ca-certificates \
bridge-utils gettext-base jq -y < /dev/null
```

### Install Oh-my-zsh

```
apt-get install zsh -y < /dev/null
use
rmod root -s /bin/zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sed -ri 's/ZSH_THEME="robbyrussell"/ZSH_THEME="agnoster"/g' .zshrc
sed -ri 's/plugins=\(git\)/plugins=\(debian apt systemd docker zsh-navigation-tools\)/g' .zshrc
echo 'export VTYSH_PAGER=more' >> /etc/zsh/zshenv
source .zshrc
```

### Add backports repo and install kernel

```
echo "deb http://ftp.debian.org/debian jessie-backports main"  >> /etc/apt/sources.list.d/debian-backports.list && apt update && apt install linux-image-amd64 linux-headers- -t jessie-backports -y < /dev/null
```

### Uninstall old kernel

```
apt remove --purge $(dpkg --list | grep linux-image-3 | cut -d " " -f 3) -y
shutdown -r now
```

### System tunning
```
cat > /etc/sysctl.d/local.conf << EOL
fs.file-max = 2097152
fs.nr_open = 2097152
net.ipv4.ip_nonlocal_bind = 1
EOL
```
```
cat > /etc/security/limits.d/local.conf <<EOL
*         hard    nofile      999999
root      hard    nofile      999999
*         soft    nofile      999999
root      soft    nofile      999999
EOL
```

### Install Sysdig

#### Sysdig for monitoring

```
curl -s https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public | apt-key add -
curl -s -o /etc/apt/sources.list.d/draios.list http://download.draios.com/stable/deb/draios.list
apt-get -qq update < /dev/null
apt-get -qq -y install linux-headers-$(uname -r) < /dev/null || kernel_warning
apt-get -qq -y install sysdig < /dev/null
echo "sysdig-probe" >> /etc/modules-load.d/modules.conf
modprobe sysdig-probe
```

## Instalacion de ELK stack Elasticsearch + Logstash + Kibana

### Añadir el repo

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
apt-get update
```

# Elasticsearch

Deshabilitamos la swap

```
swapoff -a
```

### Instalacion de Elasticsearch
```
apt update && apt-get install openjdk-8-jre-headless && apt-get install elasticsearch -y
```

### O en Ubuntu 18.04

```
apt update && apt-get install openjdk-11-jre-headless && apt-get install elasticsearch -y
```

### Conf:

```
vim /etc/elasticsearch/jvm.options
```
#### Modificamos memoria de la JVM

```
   ... 
   # Xms represents the initial size of total heap space
   # Xmx represents the maximum size of total heap space

   -Xms2g
   -Xmx2g
   ... 
```

#### Configuramos ES (Fich: /etc/elasticsearch/elasticsearch.yml)

```
cluster.name: elk-demo
node.name: elkdemo
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 127.0.0.1
http.port: 9200
```

### Arrancamos ES

```
systemctl enable elasticsearch.service && systemctl start elasticsearch.service && systemctl status elasticsearch.service
```

### Kibana

```
apt-get install openjdk-8-jre-headless kibana -y
```

### Config

#### /etc/kibana/kibana.yml 

```
server.port: 5601
server.host: "127.0.0.1"
server.name: "elkdemo"
elasticsearch.url: "http://127.0.0.1:9200"
```

#### Arrancar servicio

```
systemctl enable kibana.service
systemctl start kibana.service
```

### Logstash + rsyslog server

Activamos la recepción de syslog vía UDP:

```
sed -ri 's/#module(load="imudp")/module(load="imudp")/g' /etc/rsyslog.conf
sed -ri 's/#input(type="imudp" port="514")/input(type="imudp" port="514")/g' /etc/rsyslog.conf
```

Creamos el template json para qye syslog lo saque hacia logstash con él

```
cat >  /etc/rsyslog.d/01-json-template.conf < EOF
template(name="json-template"
  type="list") {
    constant(value="{")
      constant(value="\"@timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
      constant(value="\",\"@version\":\"1")
      constant(value="\",\"message\":\"")     property(name="msg" format="json")
      constant(value="\",\"sysloghost\":\"")  property(name="hostname")
      constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
      constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
      constant(value="\",\"programname\":\"") property(name="programname")
      constant(value="\",\"procid\":\"")      property(name="procid")
    constant(value="\"}\n")
}
EOF
```

Hacemos que mande todo a logstash
```
/etc/rsyslog.d/60-output.conf

```

```
/etc/logstash/conf.d/logstash.conf
# This input block will listen on port 10514 for logs to come in.
# host should be an IP on the Logstash server.
# codec => "json" indicates that we expect the lines we're receiving to be in JSON format
# type => "rsyslog" is an optional identifier to help identify messaging streams in the pipeline.
input {
  udp {
    host => "XXXXXXXX"
    port => 10514
    codec => "json"
    type => "rsyslog"
  }
}
# This is an empty filter block.  You can later add other filters here to further process
# your log lines
filter { }
# This output block will send all events of type "rsyslog" to Elasticsearch at the configured
# host and port into daily indices of the pattern, "rsyslog-YYYY.MM.DD"
output {
  if [type] == "rsyslog" {
    elasticsearch {
      hosts => [ "XXXXXXXX:9200" ]
    }
  }
}
```

Test si todo está escuchando
```
ss -putan | grep 514     
udp    UNCONN     0      0         *:514                   *:*                   users:(("rsyslogd",pid=26165,fd=6))
udp    UNCONN     0      0      XXXXXXXX:10514                 *:*                   users:(("java",pid=25910,fd=85))
udp    UNCONN     0      0        :::514                  :::*                   users:(("rsyslogd",pid=26165,fd=7))
```


```
systemctl enable logstash
systemctl start logstash
```

Si queremos que metricbeat nos saque metricas de logstash:

```
metricbeat modules enable logstash
vim /etc/metricbeat/modules.d/logstash.yml
systemctl restart metricbeat.service
```

#Filebeat & Metricbeat

```
apt-get install filebeat metricbeat -y


sed -ri 's/localhost:9200/es01:9200/g' /etc/filebeat/filebeat.yml
sed -ri 's/localhost:9200/es01:9200/g' /etc/metricbeat/metricbeat.yml
sed -ri 's/localhost:5601/es01:5601/g' /etc/filebeat/filebeat.yml
sed -ri 's/localhost:5601/es01:5601/g' /etc/metricbeat/metricbeat.yml
sed -ri 's/127.0.0.1:5601/es01:5601/g' /etc/filebeat/filebeat.yml
sed -ri 's/127.0.0.1:5601/es01:5601/g' /etc/metricbeat/metricbeat.yml


#Activamos los modulos que queramos:
metricbeat modules list
metricbeat modules enable nginx
vim /etc/metricbeat/metricbeat.yml #editar tags
systemctl enable metricbeat ; systemctl start metricbeat
vim /etc/filebeat/filebeat.yml
filebeat modules list
filebeat modules enable system
systemctl enable filebeat ; systemctl start filebeat

```

apt-get install packetbeat -y
sed -ri 's/localhost:9200/es01:9200/g' /etc/packetbeat/packetbeat.yml
sed -ri 's/localhost:5601/es01:5601/g' /etc/packetbeat/packetbeat.yml
sed -ri 's/127.0.0.1:5601/es01:5601/g' /etc/packetbeat/packetbeat.yml
systemctl enable packetbeat ; systemctl start packetbeat

## Configurar indices y dhasboard

En un host que tenga la conf de metricbeat y filebeat puesta a:

```
setup.dashboards.enabled: true
setup.kibana:
  host: "XXXXXXX:5601"

```

```
metricbeat setup
filebeat setup
```


