FROM centos:7.6.1810

LABEL author="IVAN DU" build-date="2019-04-14" name="Toolbox"

RUN yum -y install epel-release && \
    yum -y install vim iproute tcpdump stress-ng  \ 
    bonnie++ strace iperf lsof ethtool sysbench \ 
    hdparm nmap && \
    yum clean all

CMD [ "/bin/bash" ]