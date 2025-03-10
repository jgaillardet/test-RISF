FROM nginx:1.27.4

RUN chown -R nginx:nginx /etc/nginx/

COPY --chown=nginx files/conf/nginx.conf /etc/nginx/nginx.conf
COPY --chown=nginx files/www/index.html /usr/share/nginx/html

USER nginx

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
