FROM wordpress:php8.2-apache 
RUN apt-get update \ 
    && apt-get install -y --no-install-recommends \ 
        libpq-dev \ 
        curl \ 
    && docker-php-ext-install pdo_pgsql pgsql \ 
    && rm -rf /var/lib/apt/lists/* 
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \ 
    CMD curl -f http://localhost/ || exit 1 
