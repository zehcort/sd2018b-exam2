FROM centos:7

## Package  installation
RUN yum install httpd


## Port configuration for Postgres

EXPOSE 80
CMD sudo systemctl start httpd
CMD systemctl enable httpd

