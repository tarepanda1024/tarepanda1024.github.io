---

title: telegraf

## tags:



config: at`/etc/telegraf/telegraf.conf`





```
https://github.com/influxdata/influxdata-docker

docker pull influxdb

docker network create influxdb

docker run -d  -p 8086:8086 -v /home/lxd/data/docker/influxdb:/var/lib/influxdb  --net=influxdb \
 -e INFLUXDB_USER=telegraf -e INFLUXDB_USER_PASSWORD=123456  -e PRE_CREATE_DB="telegraf" --name influxdb influxdb

docker pull telegraf

docker run -d --name=telegraf \
      --net=influxdb \
      -v /home/lxd/data/docker/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro \
      telegraf

docker run -d -p 8888:8888 \
      --net=influxdb  \
      -v /home/lxd/data/docker/chronograf:/var/lib/chronograf \
      --name   chronograf chronograf --influxdb-url=http://influxdb:8086 
```

```
docker pull chronograf

docker pull kapacitor
```






