FROM containers.intersystems.com/intersystems/webgateway:2024.1

USER root

COPY config/apache/iris.conf /etc/apache2/sites-available/iris.conf

RUN a2ensite iris.conf \
    && a2dissite 000-default.conf \
    && apachectl configtest
    