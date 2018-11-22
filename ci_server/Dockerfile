## Using a Docker image that have Python 3.6

FROM python:3.6

## Upgrading pip and installing required libraries

RUN pip3.6 install --upgrade pip
RUN pip3.6 install connexion[swagger-ui]
RUN pip3.6 install --trusted-host pypi.python.org  docker[tls]

## Copying the content of the directory in the container folder

COPY ./handler_endpoint /handler_endpoint
RUN ["chmod", "+x", "/handler_endpoint/deploy.sh"]

## Deploying app

WORKDIR /handler_endpoint
CMD ./deploy.sh
