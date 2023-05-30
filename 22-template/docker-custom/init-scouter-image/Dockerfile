FROM registry.redhat.io/ubi8/ubi

LABEL maintainer=Timegate

WORKDIR /opt

USER 0

ADD scouter ./scouter 

RUN chmod 777 ./scouter
