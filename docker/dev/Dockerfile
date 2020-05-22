FROM jc21/alpine-nginx-full:node
LABEL maintainer="Jamie Curnow <jc@jc21.com>"

ENV S6_LOGGING=0
ENV SUPPRESS_NO_CONFIG_WARNING=1
ENV S6_FIX_ATTRS_HIDDEN=1

RUN echo "fs.file-max = 65535" > /etc/sysctl.conf \
	&& rm -rf /etc/nginx \
	&& apk update \
	&& apk add python2 certbot jq \
	&& rm -rf /var/cache/apk/*

# Task
RUN cd /usr \
	&& curl -sL https://taskfile.dev/install.sh | sh \
	&& cd /root

COPY rootfs /
RUN rm -f /etc/nginx/conf.d/production.conf

# s6 overlay
RUN curl -L -o /tmp/s6-overlay-amd64.tar.gz "https://github.com/just-containers/s6-overlay/releases/download/v1.22.1.0/s6-overlay-amd64.tar.gz" \
	&& tar -xzf /tmp/s6-overlay-amd64.tar.gz -C /

RUN mkdir -p /opt/luarocks-build && cd /opt/luarocks-build && apk add readline-dev perl pcre-dev openssl-dev zlib-dev
RUN wget -O /opt/luarocks-build/luarocks-3.3.1.tar.gz http://luarocks.github.io/luarocks/releases/luarocks-3.3.1.tar.gz && wget -O /opt/luarocks-build/lua-5.1.5.tar.gz http://www.lua.org/ftp/lua-5.1.5.tar.gz
RUN mkdir -p /opt/openresty-build && cd /opt/openresty-build && wget -O /opt/openresty-build/openresty-1.15.8.3.tar.gz https://openresty.org/download/openresty-1.15.8.3.tar.gz
RUN cd /opt/openresty-build && tar -xvf openresty-1.15.8.3.tar.gz && cd /opt/openresty-build/openresty-1.15.8.3 && ./configure --prefix=/etc/nginx \
	--sbin-path=/usr/sbin/nginx \
	--modules-path=/usr/lib/nginx/modules \
	--conf-path=/etc/nginx/nginx.conf \
	--error-log-path=/var/log/nginx/error.log \
	--http-log-path=/var/log/nginx/access.log \
	--pid-path=/var/run/nginx.pid \
	--lock-path=/var/run/nginx.lock \
	--http-client-body-temp-path=/var/cache/nginx/client_temp \
	--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
	--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
	--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
	--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
	--user=nginx \
	--group=nginx \
	--with-compat \
	--with-threads \
	--with-http_addition_module \
	--with-http_auth_request_module \
	--with-http_dav_module \
	--with-http_flv_module \
	--with-http_gunzip_module \
	--with-http_gzip_static_module \
	--with-http_mp4_module \
	--with-http_random_index_module \
	--with-http_realip_module \
	--with-http_secure_link_module \
	--with-http_slice_module \
	--with-http_ssl_module \
	--with-http_stub_status_module \
	--with-http_sub_module \
	--with-http_v2_module \
	--with-mail \
	--with-mail_ssl_module \
	--with-stream \
	--with-stream_realip_module \
	--with-stream_ssl_module \
	--with-stream_ssl_preread_module -j2 && make -j2 && make install
RUN cd /opt/luarocks-build && tar -zxf lua-5.1.5.tar.gz && tar zxpf luarocks-3.3.1.tar.gz
RUN cd /opt/luarocks-build/lua-5.1.5 && make linux test && make install
RUN cd /opt/luarocks-build/luarocks-3.3.1 && ./configure && make && make install
RUN apk add unzip && luarocks install lua-resty-openidc && luarocks install lua-cjson

EXPOSE 80
EXPOSE 81
EXPOSE 443

CMD [ "/init" ]

HEALTHCHECK --interval=5s --timeout=3s CMD /bin/check-health
