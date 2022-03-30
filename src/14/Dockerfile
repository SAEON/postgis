FROM postgis/postgis:14-3.2

RUN apt-get update -y
RUN apt-get install postgis -y
RUN apt-get clean
RUN rm -rf /var/cache/apt/lists