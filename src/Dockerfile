ARG POSTGIS_VERSION
FROM postgis/postgis:$POSTGIS_VERSION

RUN apt-get update -y
RUN apt-get install postgis -y
RUN apt-get clean
RUN rm -rf /var/cache/apt/lists