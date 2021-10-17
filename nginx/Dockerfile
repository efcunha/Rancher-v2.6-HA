FROM nginx:alpine
 
COPY nginx.conf /etc/nginx/nginx.conf
COPY tls.crt /etc/nginx/conf.d/tls.crt
COPY tls.key /etc/nginx/conf.d/tls.key

RUN apk update && apk add bash
