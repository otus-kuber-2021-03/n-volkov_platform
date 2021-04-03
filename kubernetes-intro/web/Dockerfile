FROM nginx:1.18.0-alpine
MAINTAINER Nikolay Volkov <blackwolf292@gmail.com>
RUN apk --no-cache add shadow && \
    usermod -u 1001 nginx && \
    mkdir /app && \
    chown nginx:nginx /app
COPY app.conf /etc/nginx/conf.d/
EXPOSE 8000
