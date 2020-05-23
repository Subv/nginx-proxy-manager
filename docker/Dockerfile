# This is a Dockerfile intended to be built using `docker buildx`
# for multi-arch support. Building with `docker build` may have unexpected results.

# This file assumes that the frontend has been built using ./scripts/frontend-build

FROM --platform=${TARGETPLATFORM:-linux/amd64} jc21/alpine-nginx-full:node

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG BUILD_VERSION
ARG BUILD_COMMIT
ARG BUILD_DATE

ENV SUPPRESS_NO_CONFIG_WARNING=1
ENV S6_FIX_ATTRS_HIDDEN=1
ENV NODE_ENV=production

RUN echo "fs.file-max = 65535" > /etc/sysctl.conf \
	&& rm -rf /etc/nginx \
	&& apk update \
	&& apk add python2 certbot jq \
	&& rm -rf /var/cache/apk/*

ENV NPM_BUILD_VERSION="${BUILD_VERSION}" NPM_BUILD_COMMIT="${BUILD_COMMIT}" NPM_BUILD_DATE="${BUILD_DATE}"

# s6 overlay
COPY scripts/install-s6 /tmp/install-s6
RUN /tmp/install-s6 "${TARGETPLATFORM}" && rm -f /tmp/install-s6

EXPOSE 80
EXPOSE 81
EXPOSE 443

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

COPY docker/rootfs      /
ADD backend             /app
ADD frontend/dist       /app/frontend

WORKDIR /app
RUN yarn install

# Remove frontend service not required for prod, dev nginx config as well
RUN rm -rf /etc/services.d/frontend RUN rm -f /etc/nginx/conf.d/dev.conf

VOLUME [ "/data", "/etc/letsencrypt" ]
CMD [ "/init" ]

HEALTHCHECK --interval=5s --timeout=3s CMD /bin/check-health
