FROM nginx

# ADD STATIC BLOG CONTENTS
COPY public /usr/share/nginx/html

# CHANGE DEFAULT CONFS
COPY nginx/*.conf /etc/nginx/conf.d/
RUN rm /etc/nginx/conf.d/default.conf

# CREATE VOLUME FOLDER FOR CERTIFICATE
RUN mkdir -p /letsencrypt
VOLUME [ "/letsencrypt" ]