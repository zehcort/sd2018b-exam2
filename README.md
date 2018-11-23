# sd2018b-exam2
**Icesi University**  
**Course:** Distributed Systems   
**Professor:** Daniel Barragán C.  
**Subject:** Artifact building for Continous Delivery  
**Email:** daniel.barragan at correo.icesi.edu.co  
**Student:** Juan Camilo Swan.  
**Student ID:** A00054620  
**Git URL:** https://github.com/juanswan13/sd2018b-exam2.git  

## Expected results
* Develop artifact automatic building for Continous Delivery  
* Use libraries of programming languages to perform specific tasks   
* Diagnose and execute the needed actions to achieve a stable infrastructure  

## Used technologies
* Docker  
* CentOS7 Box
* Github Repository
* Python3
* Python3 Libraries: Flask, Fabric
* Ngrok  

## Infrastructure diagram  
The desired infrastructure to deploy consists in three Docker Containers and one Docker Client with the following settings:

* Python:3.6-slim CI Server: this CT has a Flask application with and endpoint using RESTful architecture best practices. The endpoint has the following logic:   
  * A Webhook attached to a Pull Request triggers the endpoint.  
  * The endpoint reads the Pull Request content and validates if the PR is mergedd  
  * If merged, via the Docker Python SDK, the endpoint runs the required commands to build the Docker Artifact and push it to the local registry.  
* wernight/ngrok Ngrok: this CT creates a temporary public domain name to expose the CI Server's endpoint.  
* registry Registry: this CT is a private local registry where the created artifacts will be pushed.  
* Windows 10 Home Docker Client: this Docker Client will be used to pull the private registry's artifacts.


![][1]  
**Figure 1**. Deploy Diagram  

## Introduction  

The current branch contains two key elements to deploy the infrastructure. The first one is the docker-compose.yml. This file contains the provisioning required for each CT. The second one is the ci_server folder that contains the Dockerfile and python script to build the CI Server Docker image.

### docker-compose.yml:
the docker-compose.yml contains three services, that will be described above  

* **Registry:** This service refers to the private local Registry for Docker images that we are creating. It uses the a Docker image called Registry that Docker already provide to create this type of servers, it is attached to the port 443. It also has two self-signed SSL certificates created with OpenSSL. These certificates  allow the server to be secure and that clients can trust in it. they are located at ./certs.
```
swan-registry.icesi.edu.co:
        image: registry:2
        container_name: swan-registry.icesi.edu.co
        volumes:
            - './docker_data/certs:/certs'
        environment:
            - 'REGISTRY_HTTP_ADDR=0.0.0.0:443'
            - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt
            - REGISTRY_HTTP_TLS_KEY=/certs/domain.key
        ports:
            - '443:443'
```
* **ngrok:** This refers to the Ngrok service. It uses the wernight/ngrok Docker image, attached to the 4040 port and it's linked to ci_server in its 80 port to expose the CI Server's endpoint.
```
    ngrok:
        image: wernight/ngrok
        ports:
            - 0.0.0.0:4040:4040
        links:
            - ci_server_swan
        environment:
            NGROK_PORT: ci_server_swan:8080
```
* **ci_server:** The ci_server contains two components to be deployed. First, it has the Dockerfile to build a proper image and the endpoind that is develop in python 3 and using the conexion libraries.
The endpoint manage the GitHub WebHook JSON and deside if the PR made in the repository was a merge, if not it return that it was not a merge, but if the repositry was merged the endpoint search for the new images added to the repository, build them and push the images to the Registry server. This endpoint can manage the PR of many images by using a well organiced JSON and the diferent images created each in one diferent folder.
* Python: handlers.py:
```
`import os
import logging
import requests
import json
import docker
from flask import Flask, request, json

def hello():
    result = {'command_return': 'Hello'}
    return result

def repository_merged():
    retorno = ""
    post_json_data = request.get_data()
    string = str(post_json_data, 'utf-8')
    jsonFile = json.loads(string)
    itWasMerged = jsonFile["pull_request"]["merged"]
    domain = 'swan-registry.icesi.edu.co:443'
    if itWasMerged:
        pull_id = jsonFile["pull_request"]["head"]["sha"]
        json_image_url = 'https://raw.githubusercontent.com/juanswan13/sd2018b-exam2/' + pull_id + '/images.json'
        json_image_response = requests.get(json_image_url)
        images_json = json.loads(json_image_response.content)
        for service in images_json["images"]:
            service_name = service["service_name"]
            image_type = service["type"]
            image_version = service["version"]
            if image_type == 'Docker':
                image_url = 'https://raw.githubusercontent.com/juanswan13/sd2018b-exam2/' + pull_id + '/' + service_name + '/Dockerfile'
                file_response=requests.get(image_url)
                file = open("Dockerfile","w")
                file.write(str(file_response.content, 'utf-8'))
                file.close()
                image_tag = domain + '/' + service_name + ':' + image_version
                client = docker.from_env()
                client.images.build(path="./", tag=image_tag)
                client.images.push(image_tag)
                client.images.remove(image=image_tag, force=True)
                retorno = image_tag + " - " + retorno
            elif image_type == 'AMI':
                image_url = 'https://raw.githubusercontent.com/juanswan13/sd2018b-exam2/' + pull_id + '/' + service_name + '/AMI'
                file_response=requests.get(image_url)
                file = open("AMI","w")
                file.write(str(file_response.content, 'utf-8'))
                file.close()
                image_tag = domain + '/' + service_name + ':' + image_version

                retorno = image_tag + " - " + retorno
            else:
                result = {'command_return': 'ERROR: JSON does not have a correct format.'}
        result = {'command_return': retorno}
    else:
        result = {'command_return': 'Pull request was not merged'}
    return result
 ```
 * Dockerfile:
  ```
  FROM python:3.6


RUN pip3.6 install --upgrade pip
RUN pip3.6 install connexion[swagger-ui]
RUN pip3.6 install --trusted-host pypi.python.org  docker[tls]


COPY ./handler_endpoint /handler_endpoint


WORKDIR /handler_endpoint

CMD ./deploy.sh
```
* Data in docker-compose:
```
    ci_server_swan:
        build: ci_server
        container_name: ci_server_swan
        environment:
            - 'CI_SERVER_HTTP_ADDR=0.0.0.0:8080'
        ports:
            - '8080:8080'
```
## Deployment
Run the following command in the repository folder:
```
docker-compose up --build
```
This command will start the build of each of the services specified previously. It has the --build parameter to run it and see the console of what are the services doing. When the services are deployed, you will get an output like this:  

![][2]  
**Figure 2**. Docker Containers created

Now, open Ngrok by opening Web UI in the browser. here go to the url 0.0.0.0:4040.  

![][3]  
**Figure 3**. Ngrok running

Finally, let's create the Github webhook. In the repository, go to *Settings -> Webhooks*. Add a webhook and put in the Payload URL the URL that ngrok proviudes and the endpoint url.

![][4]  
**Figure 4**. Github Webhook working

## Demonstration  

To show that the exam works well, two more branches were created, one named jcswan/develop where i create a JSON and two folders, and inside each folder one vagrant file. this JSON format is the following:
```

{
  "images": [
    {
      "service_name": "postgresql",
      "version": "1.0",
      "type": "Docker"
    },
    {
      "service_name": "httpd",
      "version": "1.0",
      "type": "Docker"
    }
  ]
}
```
So is easy to recognize that two images are going to be generated. The next step is to make the pull request of this branch to the branch of develop. this will make the endpoint to activate but it will say that the repository was not merged; this is because at this point we haven't merged it.

![][5]  
**Figure 5**. PR Created.

![][6]  
**Figure 6**. Webhook working.

Then we go again to the PR and click to the buttom **merge**. this will make the webhook to activate again and this time the endpoint will recognize that the PR was merged so it will catch the images, build them and push them to the registry.

![][7]  
**Figure 7**. PR Merged.

![][8]  
**Figure 8**. CI Server answering.

![][9]  
**Figure 9**. OK-200 response from CI. (and the name of the services and the registry domain can be seen in the return).

![][10]  
**Figure 10**. Previous the PR was merged the registry has no images.

![][11]  
**Figure 11**. Now we can pull one image of those that the endpoint generate.

## References  
* https://docker-py.readthedocs.io/en/stable/
* https://www.learnitguide.net/2018/07/create-your-own-private-docker-registry.html
* https://www.youtube.com/watch?v=SEpR35HZ_hQ
* https://forums.docker.com/t/running-an-insecure-registry-insecure-registry/8159/5
* https://hub.docker.com/_/registry/
* https://hub.docker.com/_/docker/
* https://github.com/juanswan13/sd2018b-exam1/tree/jswan/exam1
* https://raw.githubusercontent.com/abc1196/sd2018b-exam2/abueno/exam2/README.md

[1]: images/01_diagrama_delivery.png
[2]: images/docker-compose.png
[3]: images/ngrok-running.png
[4]: images/Webhook-creation.png  
[5]: images/Pr-Created.png
[6]: images/action_generated_by_pr_generation.png
[7]: images/PR-Merged.png
[8]: images/ci-server-answering.png
[9]: images/ok-response-from-ci.png
[10]: images/no-images-in-mirror.png
[11]: images/image-pulled.png
