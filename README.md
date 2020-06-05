# ELK Ubiquiti Unifi USG and CloudKey with a side of Pi-hole

This all assumes you already have Elastic Stack up and running.  
If you don't, I have included my docker-compose.yml that I used to run a single Elasticsearch node, Kibana and Logstash

Credit for this work goes to others, I simple modified/modernized their work! 
* Beats compiling/config for USG - [caglar10ur](https://github.com/caglar10ur/elk-unifi)
* Elastic Stack 
  - [deviantony](https://github.com/deviantony/docker-elk)
  - [Elastic](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)

## USG

On your Linux machine where you have [docker](https://www.docker.com/) and [go](https://golang.org/).

### Clone the repository
```bash
git clone https://github.com/owentl/elk-unifi.git ~/elk-unifi
```

### Build beats for MIPS64 and put them under ~/elk-unifi/
```bash

mkdir -p ~/go/src/github.com/elastic/

git clone -b v7.6.2 https://github.com/elastic/beats.git ~/go/src/github.com/elastic/beats
pushd  ~/go/src/github.com/elastic/beats/filebeat
GOOS=linux GOARCH=mips64 go build -o ~/elk-unifi/filebeat/filebeat
popd

pushd  ~/go/src/github.com/elastic/beats/metricbeat
GOOS=linux GOARCH=mips64 go build -o ~/elk-unifi/metricbeat/metricbeat
popd
```

Now we need to setup the files to report back to your Elastic Stack.  Change the references to 192.168.1.208 to the IP of your Elastic Stack

### Edit the filebeats.yml 
```bash
vi ~/elk-unifi/filebeats/filebeats.yml
```
### Edit the filebeats.yml 
```bash
vi ~/elk-unifi/metricbeats/metricbeats.yml
```


### Copy ~/elk-unifi to USG
```bash
scp -pr ~/elk-unifi/ admin@192.168.1.1:
```

### SSH to USG
```bash
ssh 192.168.1.1 -l admin
```

### Register filebeat template and dashboard
```bash
cd elk-unifi/filebeat
./filebeat setup --path.config /home/admin/elk-unifi/filebeat/
```

### Test filebeat 
```bash
./filebeat --path.config /home/admin/elk-unifi/filebeat/
```

### Register metricbeat template and dashboard
```bash
cd elk-unifi/metricbeats
./metricbeat setup --path.config /home/admin/elk-unifi/metricbeat/
```

### Test metricbeat
```bash
./metricbeat -e --path.config /home/admin/elk-unifi/metricbeat/
```

### Start beats
```bash
nohup /home/admin/elk-unifi/filebeat/filebeat run -c /home/admin/elk-unifi/filebeat/filebeat.yml >/dev/null 2>&1 &
nohup /home/admin/elk-unifi/metricbeat/metricbeat run -c /home/admin/elk-unifi/metricbeat/metricbeat.yml >/dev/null 2>&1 &
```

## CloudKey

### Build beats for ARMv7 and put them under ~/elk-unifi/ (assumes previous git pull, etc were done)
```bash
#preserve mips based filebeat
mv ~/elk-unifi/filebeat/filebeat ~/elk-unifi/filebeat/filebeat-mips
pushd  ~/go/src/github.com/elastic/beats/filebeat
GOOS=linux GOARCH=arm GOARM=7 go build -o ~/elk-unifi/filebeat/filebeat
popd

#preserve mips based metricbeat
mv ~/elk-unifi/metricbeat/metricbeat ~/elk-unifi/metricbeat/metricbeat-mips
pushd  ~/go/src/github.com/elastic/beats/metricbeat
GOOS=linux GOARCH=arm GOARM=7 go build -o ~/elk-unifi/metricbeat/metricbeat
popd
```

### Copy ~/elk-unifi to CloudKey
```bash
scp -pr ~/elk-unifi/ root@192.168.1.2:
```

### SSH to CloudKey
```bash
ssh root@192.168.1.2
```
### Update path variable for both metricbeat and filebeat in YAML
Edit each yml file and set the correct path.  On USG it is /home/admin and CloudKey it is /root/

### Add/Enable nginx_status
By default the cloudkey Plus will forward all http connections over to https.  
I am not an nginx person so I presume there is a better way to do than this, but it works for me

Edit the unifi site:
```
/etc/nginx/sites-available/unifi-management-portal 
```
and add the below after the ```server_tokens off;``` line in the first "server" stanza

```bash
location /nginx_status {
        stub_status on;
        allow 127.0.0.1;        #only allow requests from localhost
        deny all;               #deny all other hosts
  }
  ```
  Comment out the line: 
  ```
  #return 302 https://$host$request_uri;
  ```
  
  Now restart nginx for these changes to go into effect
  
  ```
  systemctl restart nginx
  ```

### Setup system for beats
```bash
mkdir /var/log/filebeat
mkdir /var/log/metricbeat
chmod 700 /var/log/filebeat
chmod 700 /var/log/metricbeat
```

### Enable nginx and mongodb for both filebeat and metricbeat
```bash
cd elk-unifi/filebeat
./filebeat --path.config /root/elk-unifi/filebeat/ modules enable nginx
./filebeat --path.config /root/elk-unifi/filebeat/ modules enable mongodb
cd elk-unifi/metricbeat
./metricbeat --path.config /root/elk-unifi/metricbeat/ modules enable nginx
./metricbeat --path.config /root/elk-unifi/metricbeat/ modules enable mongodb
```

### Register filebeat template and dashboard
```bash
cd elk-unifi/filebeat
./filebeat setup --path.config /root/elk-unifi/filebeat/
```
### Configure systemctl scripts
```bash
mv /root/elk-unifi/filebeat/cloudkey/filebeat.service /lib/systemd/system/filebeat.service
mv /root/elk-unifi/metricbeat/cloudkey/metricbeat.service /lib/systemd/system/metricbeat.service
systemctl daemon-reload
```

## Pi-Hole
Since we already have a compiled ARM binary we can now use that on the raspberry pi.  Follow the same steps for the Cloud Key config and get Filebeat running there.

If you are using [Elk-Hole](https://github.com/nin9s/elk-hole) config you will need to also have Logstash configured and you can follow the steps there.
