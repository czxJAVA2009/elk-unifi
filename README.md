# ELK Ubiquiti USG and CloudKey

## USG

On your Linux machine where you have [docker](https://www.docker.com/) and [go](https://golang.org/).

### Clone the repository
```bash
git clone https://github.com/owentl/elk-usg.git ~/elk-usg
```

### Build beats for MIPS64 and put them under ~/elk-usg/
```bash

mkdir -p ~/go/src/github.com/elastic/

git clone -b v7.6.2 https://github.com/elastic/beats.git ~/go/src/github.com/elastic/beats
pushd  ~/go/src/github.com/elastic/beats/filebeat
GOOS=linux GOARCH=mips64 go build -o ~/elk-usg/filebeat/filebeat
popd

pushd  ~/go/src/github.com/elastic/beats/metricbeat
GOOS=linux GOARCH=mips64 go build -o ~/elk-usg/metricbeat/metricbeat
popd
```

Now we need to setup the files to report back to your Elastic Stack.  Change the references to 192.168.1.208 to the IP of your Elastic Stack

### Edit the filebeats.yml 
```bash
vi ~/elk-usg/filebeats/filebeats.yml
```
### Edit the filebeats.yml 
```bash
vi ~/elk-usg/metricbeats/metricbeats.yml
```


### Copy ~/elk-usg to USG
```bash
scp -pr ~/elk-usg/ admin@192.168.1.1:
```

### SSH to USG
```bash
ssh 192.168.1.1 -l admin
```

### Register filebeat template and dashboard
```bash
cd elk-usg/filebeat
./filebeat setup --path.config /home/admin/elk-usg/filebeat/
```

### Test filebeat 
```bash
./filebeat --path.config /home/admin/elk-usg/filebeat/
```

### Register metricbeat template and dashboard
```bash
cd elk-usg/metricbeats
./metricbeat setup --path.config /home/admin/elk-usg/metricbeat/
```

### Test metricbeat
```bash
./metricbeat -e --path.config /home/admin/elk-usg/metricbeat/
```

### Start beats
```bash
nohup /home/admin/elk-usg/filebeat/filebeat run -c /home/admin/elk-usg/filebeat/filebeat.yml >/dev/null 2>&1 &
nohup /home/admin/elk-usg/metricbeat/metricbeat run -c /home/admin/elk-usg/metricbeat/metricbeat.yml >/dev/null 2>&1 &
```

## CloudKey

### Build beats for ARMv7 and put them under ~/elk-usg/ (assumes previous git pull, etc were done)
```bash
#preserve mips based filebeat
mv ~/elk-usg/filebeat/filebeat ~/elk-usg/filebeat/filebeat-mips
pushd  ~/go/src/github.com/elastic/beats/filebeat
GOOS=linux GOARCH=arm GOARM=7 go build -o ~/elk-usg/filebeat/filebeat
popd

#preserve mips based metricbeat
mv ~/elk-usg/metricbeat/metricbeat ~/elk-usg/metricbeat/metricbeat-mips
pushd  ~/go/src/github.com/elastic/beats/metricbeat
GOOS=linux GOARCH=arm GOARM=7 go build -o ~/elk-usg/metricbeat/metricbeat
popd
```

### Copy ~/elk-usg to CloudKey
```bash
scp -pr ~/elk-usg/ root@192.168.1.2:
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
cd elk-usg/filebeat
./filebeat --path.config /root/elk-usg/filebeat/ modules enable nginx
./filebeat --path.config /root/elk-usg/filebeat/ modules enable mongodb
cd elk-usg/metricbeat
./metricbeat --path.config /root/elk-usg/metricbeat/ modules enable nginx
./metricbeat --path.config /root/elk-usg/metricbeat/ modules enable mongodb
```

### Register filebeat template and dashboard
```bash
cd elk-usg/filebeat
./filebeat setup --path.config /root/elk-usg/filebeat/
```
### Configure systemctl scripts
```bash
mv /root/elk-usg/filebeat/cloudkey/filebeat.service /lib/systemd/system/filebeat.service
mv /root/elk-usg/metricbeat/cloudkey/metricbeat.service /lib/systemd/system/metricbeat.service
systemctl daemon-reload
```

## Pi-Hole
Since we already have a compiled ARM binary we can now use that on the raspberry pi.  Follow the same steps for the Cloud Key config and get Filebeat running there.

If you are using [Elk-Hole](https://github.com/nin9s/elk-hole) config you will need to also have Logstash configured and you can follow the steps there.
