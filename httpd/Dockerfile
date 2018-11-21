FROM centos:7

## Package installation

RUN yum install httpd

## Port configuration and enabling service

EXPOSE 80
CMD sudo systemctl start httpd
CMD systemctl enable httpd

