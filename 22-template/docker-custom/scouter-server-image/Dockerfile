FROM docker.io/scouterapm/scouter-paper:2.6.4 as builder

FROM docker.io/scouterapm/scouter-server:2.17.1

LABEL maintainer=Timegate

WORKDIR /opt

USER 0

COPY --from=builder . /extweb
