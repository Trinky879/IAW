FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y apache2 curl && rm -rf /var/lib/apt/lists/*
RUN a2dismod mpm_prefork mpm_worker || true && a2enmod mpm_event \
    && sed -i 's/Listen 80/Listen 8016/g' /etc/apache2/ports.conf \
    && a2dissite 000-default.conf
COPY sites/ /etc/apache2/sites-available/
COPY www/ /var/www/
RUN a2ensite 00-afornell-catchall.conf 10-afornell-publica.conf 20-afornell-interna.conf
EXPOSE 8016
CMD ["apachectl", "-D", "FOREGROUND"]
