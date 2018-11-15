FROM centos/python-36-centos7

RUN pip3.6 install --upgrade pip
RUN pip3.6 install connexion[swagger-ui]
RUN pip3.6 install Docker

COPY ./handler_endpoint /handler_endpoint

WORKDIR /handler_endpoint

CMD ./deploy.sh
