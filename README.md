### sd2018b-exam2

**Universidad Icesi**  
**Curso:** Sistemas Distribuidos  
**Profesor:** Daniel Barragán C.  
**Tema:** Construcción de artefactos para entrega continua  
**Email:** daniel.barragan at correo.icesi.edu.co  
**Estudiante:** Luis Alejandro Tróchez Arredondo  
**Código:** A00054648  
**URL Git:** https://github.com/zehcort/sd2018b-exam2  

### Objetivos
* Realizar de forma autómatica la generación de artefactos para entrega continua
* Emplear librerías de lenguajes de programación para la realización de tareas específicas
* Diagnosticar y ejecutar de forma autónoma las acciones necesarias para corregir fallos en
la infraestructura

### Tecnlogías sugeridas para el desarrollo del examen
* Docker
* Box del sistema operativo CentOS7
* Repositorio Github
* Python3
* Librerias Python3: Flask, Connexion, Docker
* Ngrok

### Descripción
Para la realización de la actividad tener en cuenta lo siguiente:

* Crear un Fork del repositorio sd2018b-exam2 y adicionar las fuentes de un microservicio
de su elección.
* Alojar en su fork un archivo Dockerfile para la construcción de un artefacto tipo Docker a
partir de las fuentes de su microservicio.

Deberá probar y desplegar los siguientes componentes:

* Despliegue de un **registry** local de Docker para el almacenamiento de imágenes de Docker. Usar la imagen de DockerHub: https://hub.docker.com/_/registry/ . Probar que es posible descarga la imagen generada desde un equipo perteneciente a la red.

* Realizar un método en Python3.6 o superior que reciba como entrada el nombre de un servicio,
la version y el tipo (Docker ó AMI) y en su lógica realice la construcción de una imagen de Docker cuyo nombre deberá ser **service_name:version** y deberá ser publicada en el **registry** local creado en el punto anterior.

* Realizar una integración con GitHub para que al momento de realizar un **merge** a la rama
**develop**, se inicie la construcción de un artefacto tipo Docker a partir del Dockerfile y las fuentes del repositorio. Idee una estrategia para el envío del **service_name** y la **versión** a través del **webhook** de GitHub. La imagen generada deberá ser publicada en el **registry** local creado.

* Si la construcción es exitosa/fallida debera actualizarse un **badge** que contenga la palabra build y la versión del artefacto creado mas recientemente (**opcional**).

* En lugar de una máquina virtual de CentOS7 para alojar el CI server,  emplear la imagen de Docker de Docker hub para el ejecución de la API (webhook listener) y la generación del artefacto: https://hub.Docker.com/_/Docker/ (**opcional**).

![][14]
**Figura 1**. Diagrama de Arquitectura de la solución

### Actividades
1. Documento README.md en formato markdown:  
  * Formato markdown (5%)
  * Nombre y código del estudiante (5%)
  * Ortografía y redacción (5%)
2. Documentación del procedimiento para el montaje del registry (10%). Evidencias del funcionamiento (5%).
3. Documentación e implementación del método para la generación del artefacto. Incluya el código fuente en el informe. Incluya comentarios en el código donde explique cada paso realizado (20%). Evidencias del funcionamiento (5%).
4. Documentación e integración de un repositorio de GitHub junto con la generación del artefacto tipo Docker (20%). Evidencias del funcionamiento (5%).
5. El informe debe publicarse en un repositorio de github el cual debe ser un fork de https://github.com/ICESI-Training/sd2018b-exam2 y para la entrega deberá hacer un Pull Request (PR) al upstream (10%). Tenga en cuenta que el repositorio debe contener todos los archivos necesarios para el aprovisionamiento
7. Documente algunos de los problemas encontrados y las acciones efectuadas para su solución al aprovisionar la infraestructura y aplicaciones (10%)

### Introducción

Para la implementación de la solución se emplean 3 ramas: una donde estarán los servicios, otra donde esté el contenido del registry y otra vacía donde se pueda probar el merge del repositorio.

En la rama donde están los servicios están contenidos dos elementos clave para implementar la infraestructura. El primero es el docker-compose.yml, archivo donde se describe el aprovisionamiento requerido para cada CT. La segunda es la carpeta ci_server que contiene el script Dockerfile y python para crear la imagen de CI Server Docker, el cual permitirá dar el componente de Integración Continua a la solución.

### Desarrollo

A continuación se muestran algunos de los archivos utilizados en la implementación

* docker-compose.yml

```
version: '3'
services:
    registry:
        image: registry:2
        container_name: registry
        volumes:
            - ./registry/certs/:/certs
        environment:
            - 'REGISTRY_HTTP_ADDR=0.0.0.0:5000'
            - REGISTRY_HTTP_TLS_CERTIFICATE=./certs/domain.crt
            - REGISTRY_HTTP_TLS_KEY=./certs/domain.key
        ports:
            - '5000:5000'
    ci_server:
        build: ci_server
        container_name: ci_server
        environment:
            - 'CI_SERVER_HTTP_ADDR=0.0.0.0:8080'
        ports:
            - '8080:8080'
    ngrok:
        image: wernight/ngrok
        ports:
            - 0.0.0.0:4040:4040
        links:
            - ci_server
        environment:
            NGROK_PORT: ci_server:8080

```

* Dockerfile ci_server: archivo en base al cual se construye la imagen para el contenedor del servicio encargado de la Integración Continua. Aquí se utiliza una imagen con Python 3.6, ya que se requieren instalar librerias Python con facilidad.

```
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

```

* handlers.py: aquí se ubica el método que permite recibir los PR, extraer su información y mandar los nuevos artefactos al Registry

A continuación se muestra el archivo app.py. En el se hace uso de librerías docker, flask, entre otras.

```
import os
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
    domain = 'registry:5000'
    if itWasMerged:
        pull_id = jsonFile["pull_request"]["head"]["sha"]
        json_image_url = 'https://raw.githubusercontent.com/zehcort/sd2018b-exam2/' + pull_id + '/images.json'
        json_image_response = requests.get(json_image_url)
        images_json = json.loads(json_image_response.content)
        for service in images_json["images"]:
            service_name = service["service_name"]
            image_type = service["type"]
            image_version = service["version"]
            if image_type == 'Docker':
                image_url = 'https://raw.githubusercontent.com/zehcort/sd2018b-exam2/' + pull_id + '/' + service_name + '/Dockerfile'
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
                image_url = 'https://raw.githubusercontent.com/zehcort/sd2018b-exam2/' + pull_id + '/' + service_name + '/AMI'
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

* registry: este servicio hace referencia al registro local privado. Hace uso de la imagen del registry de docker y se le proporciona al al puerto 5000. Se necesitan los certificados SSL para la seguridad del servidor. Los certificados se guardan en ./certs.

Primero se crea un directorio donde se van a almacenar los certificados de seguridad. Estos deben ser generados en el montaje de la infraestructura. Esto se realizó de la siguiente manera:

```
 openssl req -newkey rsa:4096 -nodes -sha256 -keyout `pwd`/certs/domain.key -x509 -days 365 -out `pwd`/certs/domain.crt

```

* ngrok: este utiliza la imagen de Docker wernight/ngrok. Con esta implementación se puede acceder a los enlaces generados a través de una interfaz gráfica. Aquí es donde se expone el endpoint (ci_server); se hace en el puerto 4040 y se conecta con el ci_server en su puerto 8000.

### Prueba de la infraestructura

Primero se inicia la compilación de los servicios de la siguiente manera:

```
docker-compose up --build

```

Una vez hecho esto, se muestra la infraestructura corriendo

![][1]
**Figura 2**. Consola con el servicio corriendo

Seguido de esto, se debe configurar el Webhook en Github. Esto servirá como trigger para que el endpoint funcione una vez realizado el PR de Merge

![][2]
**Figura 3**. Configuración del Webhook

Aquí se debe ajustar que se va a recibir la información en un formato JSON y una vez el repositorio un Pull Request

![][3]
**Figura 4**. Ajuste de parámetros del Webhook

Una vez listo el entorno, se crea el PR con origen en la rama vacía Develop hacia la rama Develop/exam2   

![][4]
**Figura 5**. Pull Request creado

Aquí vemos en el Ngrok el último código (200) de éxito del servicio de Integración continua

![][5]
**Figura 6**. Código exitoso en el Ngrok

Por último, podremos realizar el merge correctamente de manera manual desde el repositorio de Github

### Dificultades

La principal dificultad que se tuvo en la realización de esta solución fue lograr conectar correctamente el ci_server con el registry, ya que dependía de tener configurado el puerto por donde se conectan, además de poder implementar correctamente esto en el método handlers.py.
Adicional a esto, inicialmente también represento una dificultad la integración de las nuevas tecnologías, como Docker, a la solución y entender el funcionamiento de un contenedor.

### Referencias
* https://hub.docker.com/_/registry/
* https://hub.docker.com/_/docker/
* https://docker-py.readthedocs.io/en/stable/index.html
* https://developer.github.com/v3/guides/building-a-ci-server/
* http://flask.pocoo.org/
* https://connexion.readthedocs.io/en/latest/


[0]: images/0.png
[1]: images/1.png
[2]: images/2.png
[3]: images/3.png
[4]: images/4.png
[5]: images/5.png
