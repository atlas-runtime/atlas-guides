FROM tozd/sgx:ubuntu-bionic
RUN apt-get update  -q && apt-get upgrade -y && apt-get install make --yes --force-yes
#COPY atlas-0.1 /root/atlas
#RUN cd /root/atlas/atlas-worker; make;
#RUN cd /root/atlas/quickjs/src; make -j; make qjs_opti -j
EXPOSE 7000-7010
