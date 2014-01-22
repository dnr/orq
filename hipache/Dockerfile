FROM stackbrew/ubuntu:raring

# From https://gist.github.com/jpetazzo/6127116:
RUN echo "force-unsafe-io" > /etc/dpkg/dpkg.cfg.d/02apt-speedup
RUN echo "Acquire::http {No-Cache=True;};" > /etc/apt/apt.conf.d/no-cache

RUN apt-get -yq update
RUN apt-get -yq upgrade
RUN apt-get -yq install wget git build-essential python2.7-minimal
RUN apt-get -yq clean
RUN ln -s python2.7 /usr/bin/python
RUN useradd -U -M -u 1000 hipache

RUN wget -qO- http://nodejs.org/dist/v0.8.26/node-v0.8.26-linux-x64.tar.gz | tar -C /usr/local/ --strip-components=1 -zx
RUN apt-get -y install wget git redis-server supervisor && apt-get -y clean
RUN npm install -g hipache
RUN mkdir -p /etc/hipache
ADD ./supervisord.conf /etc/supervisor/supervisord.conf
ADD ./config.json /etc/hipache/config.json
ADD ./redis.conf /etc/redis/redis.conf
USER hipache
EXPOSE 8000
VOLUME ["/var/log/hipache"]
CMD ["supervisord", "-n"]
