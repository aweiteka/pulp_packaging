FROM centos:centos7
MAINTAINER Aaron Weitekamp <aweiteka@redhat.com>

ADD rhel-pulp.repo /etc/yum.repos.d/rhel-pulp.repo

RUN yum -y install http://dl.fedoraproject.org/pub/epel/beta/7/x86_64/epel-release-7-1.noarch.rpm

RUN yum update -y && \
    yum groupinstall -y pulp-server && \
    yum clean all

RUN yum install -y git findutils && \
    yum clean all

# install pulp_docker plugin to manage docker image repositories
#RUN git clone --single-branch https://github.com/pulp/pulp_docker.git /opt/pulp_docker
ADD python-pulp-docker-common-0.1.1-1.el7sat.noarch.rpm \
    python-pulp-docker-common-0.1.1-1.el7sat.noarch.rpm
ADD pulp-docker-plugins-0.1.1-1.el7sat.noarch.rpm \
    pulp-docker-plugins-0.1.1-1.el7sat.noarch.rpm
RUN yum install -y python-pulp-docker-common-0.1.1-1.el7sat.noarch.rpm \
    pulp-docker-plugins-0.1.1-1.el7sat.noarch.rpm

#RUN /opt/pulp_docker/manage_setup_pys.sh develop

#RUN python /opt/pulp_docker/pulp-dev.py -I

# HACK: may not be needed
#RUN touch /etc/pulp/vhosts80/blank.conf
