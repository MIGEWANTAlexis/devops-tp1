FROM httpd:2.4.52
WORKDIR /usr/local/apache2/htdocs/
COPY index.html .
WORKDIR /usr/local/apache2/conf/
COPY httpd.conf .
EXPOSE 80
CMD ["httpd-foreground"]