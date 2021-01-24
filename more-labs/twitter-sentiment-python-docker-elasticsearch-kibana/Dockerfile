# start with a base image
FROM ubuntu:18.04

MAINTAINER Hong Jee <hongjee@yahoo.com>

# initial update and install wget
RUN apt-get update -q && apt-get install -yq wget

# create a new user elasticuser
RUN groupadd --gid 1001 elasticsearch && \
    useradd --system -create-home --home-dir /home/elasticsearch --shell /bin/bash --gid elasticsearch --groups sudo --uid 1001 elasticsearch
	
# install elasticsearch
RUN [ -d /opt ] || mkdir /opt
RUN cd /opt && \
    wget -nv https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    tar zxf elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    rm -f elasticsearch-7.10.2-linux-x86_64.tar.gz && \
    mv elasticsearch-7.10.2 elasticsearch && \
	echo 'transport.host: 127.0.0.1' >> /opt/elasticsearch/config/elasticsearch.yml && \
	echo 'http.host: 0.0.0.0' >> /opt/elasticsearch/config/elasticsearch.yml && \
	chown -R elasticsearch elasticsearch && \
# install kibana
    wget -nv https://artifacts.elastic.co/downloads/kibana/kibana-7.10.2-linux-x86_64.tar.gz && \
    tar zxf kibana-7.10.2-linux-x86_64.tar.gz && \
    rm -f kibana-7.10.2-linux-x86_64.tar.gz && \
    mv kibana-7.10.2-linux-x86_64 kibana && \
#	echo 'server.host: "0.0.0.0"' >> /opt/kibana/config/kibana.yml && \
	chown -R elasticsearch kibana && \
# create entrypoint.sh 
	echo '#!/bin/bash' >> entrypoint.sh && \
	echo '/opt/elasticsearch/bin/elasticsearch -d' >> entrypoint.sh && \
	echo '/opt/kibana/bin/kibana -H 0.0.0.0 -p 5601' >> entrypoint.sh && \
	chmod a+x entrypoint.sh && \
	chown elasticsearch entrypoint.sh
	
USER elasticsearch
	
# expose ports
EXPOSE 9200 5601

# start elasticsearch
ENTRYPOINT ["/opt/entrypoint.sh"]
