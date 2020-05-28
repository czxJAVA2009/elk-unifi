# ELK USG

On your Linux machine where you have [docker](https://www.docker.com/) and [go](https://golang.org/).

## Clone the repository
```bash
git clone https://github.com/owentl/elk-usg.git ~/elk-usg
```

## Build beats for MIPS64 and put them under ~/elk-usg/
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

## Edit the filebeats.yml 
```bash
vi ~/elk-usg/filebeats/filebeats.yml
```
## Edit the filebeats.yml 
```bash
vi ~/elk-usg/metricbeats/metricbeats.yml
```


## Copy ~/elk-usg to USG
```bash
scp -pr ~/elk-usg/ admin@192.168.1.1:
```

## SSH to USG
```bash
ssh 192.168.1.1 -l admin
```

## Register filebeat template and dashboard
```bash
cd elk-usg/filebeat
./filebeat setup
```

## Test filebeat 
```bash
./filebeat
```

## Register metricbeat template and dashboard
```bash
cd elk-usg/filebeats
./metricbeat setup
```

## Test metricbeat
```bash
./metricbeat -e
```

## Start beats
```bash
nohup /home/admin/elk-usg/filebeat/filebeat run -c /home/admin/elk-usg/filebeat/filebeat.yml >/dev/null 2>&1 &
nohup /home/admin/elk-usg/metricbeat/metricbeat run -c /home/admin/elk-usg/metricbeat/metricbeat.yml >/dev/null 2>&1 &
```
