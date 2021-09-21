FROM tozd/sgx:ubuntu-bionic
#FROM ubuntu:xenial
#FROM atlas-eco_sgx:latest
# && apt-get install update-manager-core --yes -force-yes
#RUN do-release-upgrade -y

RUN apt-get update  -q && apt-get upgrade -y && apt-get install htop tmux make --yes --force-yes
COPY atlas-0.1 /root/atlas

RUN cd /root/atlas/atlas-worker; make;
RUN cd /root/atlas/quickjs/src; make -j; make qjs_opti -j

EXPOSE 7000-7010
