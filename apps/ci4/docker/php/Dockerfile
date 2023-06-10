FROM php:8.2-fpm-alpine


# Useful PHP extension installer image, copy binary into your container
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/


# Install php extensions
# exit on errors, exit on unset variables, print every command as it is executed
RUN set -eux; \
    install-php-extensions intl mysqli;    


# Enable environment variables for PHP-FPM
RUN echo "clear_env = no" >> /usr/local/etc/php-fpm.d/www.conf

# allow super user - set this if you use composer as super user at all times like in docker containers
ENV COMPOSER_ALLOW_SUPERUSER=1

# obtain composer using multi-stage build
# https://docs.docker.com/build/building/multi-stage/
COPY --from=composer:2.5 /usr/bin/composer /usr/bin/composer


# here, we are copying only composer.json and composer.lock (instead of copying the entire source)
# right before doing composer install
# this is to take advantage of docker caching mechanism, and composer install will only be executed 
# if there are changes in composer.json or composer.lock
COPY ./composer.* ./


COPY . /var/www/html

RUN composer install --optimize-autoloader --no-interaction --no-progress

RUN chmod -R 777 /var/www/html/writable/
# RUN chmod -R 777 /var/www/html/app/Database/
# RUN chmod -R 777 /var/www/html/app/Entities/
# RUN chmod -R 777 /var/www/html/app/Models/


# Install Vim
RUN apk add --no-cache vim