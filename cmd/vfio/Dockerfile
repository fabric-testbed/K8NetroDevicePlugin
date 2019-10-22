FROM centos:7

RUN yum -y install kmod

WORKDIR /

ADD vfio /vfio

CMD ["/vfio", "-v", "3", "-logtostderr"]
