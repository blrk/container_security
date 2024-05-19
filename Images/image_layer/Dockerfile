FROM ubuntu:22.04
RUN apt update -y
RUN apt install -y apache2
RUN chown -R www-data:www-data /var/www/
RUN echo "some secret" > /password.txt
RUN rm /password.txt
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_PID_FILE /var/run/apache2.pid
ADD index.html /var/www/html/
EXPOSE 80
ENTRYPOINT ["/usr/sbin/apache2ctl"]
CMD ["-D","FOREGROUND"]