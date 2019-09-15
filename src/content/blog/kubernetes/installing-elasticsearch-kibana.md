---
title: "Instalación de Elasticsearch Kibana"
date: 2019-09-10T20:02:30+02:00
draft: true
---

#Creación del cluster
Hemos creado el cluster mediante la consola web. Los parámetros utilizados
son:

- Regional
  - Región: us-east4
  - Nodos por zona: 2 (6 nodos en total)
  - Tipo de nodo: n1-standard-4 (4 vCPUs, 15GB memoria)
  - Cluster privado
  - Acceso público a la IP externa del master
  - Se habilitan 2 direcciones para permitir el acceso externo:
    - IP Corporativa de NX
    - IP Corporativa de A3G
  - Se habilita el modo Legacy de autenticación

#Inicialización de helm en el nuevo Cluster
Nos conectamos al nuevo cluster mediante el comando gcloud correspondiente e
inicializamos Helm:

    helm init

#Obtención de los charts
No vamos a utilizar los charts de Helm, sino los de Elastic. Los charts de
Kibana y Elasticsearch son mucho más flexibles y además vienen preparados
para ser integrados sin mucha configuración adicional. Además, los charts
de Helm para Elasticsearch están marcados como *Deprecated*.

El único problema que tenemos es que las versiones más actualizadas de los
charts están preparadas para realizar el despliegue de las versiones 7.x.x de
Elasticsearch y Kibana, por lo que necesitamos versiones más antiguas.

Una versión que hemos probado y funciona es la 6.5.0 de ambos charts. Parecen
totalmente compatible con la versión 6.8.3 de Kibana y Elasticsearch.

Para la obtención de los charts, procedemos de la forma habitual. Primero
configuramos el repositorio Helm de Elastic:

    helm repo add elastic https://helm.elastic.co

Una vez hecho esto, descargamos la versión 6.5.0 de ambos charts:

    helm fetch elastic/elasticsearch --version 6.5.0
    helm fetch elastic/kibana --version 6.5.0

El único inconveniente que hemos encontrado es que la versión 6.5.0 del chart
de elasticsearch no proporcion la capacidad de crear un servicio de tipo
LoadBalancer. Necesitamos editar el fichero correspondiente
(templates/service.yaml) y añadir el tipo de servicio a mano.

En cualquier caso, con el objetivo de poder tener disponibles estos charts
en un futuro, incluso en el caso de que Elastic decidiese eliminarlos de
su repositorio, los hemos descargado y añadido a nuestro repositorio
`nx-platform-terraform-ops`, con las modificaciones necesarias ya realizadas.

#Preparación de las imágenes
Una vez obtenidos los charts y realizadas las modificaciones requeridas a los
mismos, tendremos que subir las imágenes necesarias a nuestro repositorio de
GCE.

Si recordamos, el cluster que hemos creado es privado. Esto impide que podamos
acceder a recursos fuera de GCE desde los nodos del mismo, lo que incluye
los repositorios de Elastic y por tanto las imágenes utilizadas por defecto en
los charts.

La solución es crear una copia de las imágenes necesarias dentro de nuestro
repositorio GCE. Para ello tan sólo tenemos que ejecutar las siguientes órdenes:

    docker pull docker.elastic.co/elasticsearch/elasticsearch:6.8.3
    docker tag docker.elastic.co/elasticsearch/elasticsearch:6.8.3 gcr.io/nx-live-platform-handle-w-care/elasticsearch:6.8.3
    docker push gcr.io/nx-live-platform-handle-w-care/elasticsearch:6.8.3

    docker pull docker.elastic.co/kibana/kibana:6.8.3
    docker tag docker.elastic.co/kibana/kibana:6.8.3 gcr.io/nx-live-platform-handle-w-care/kibana:6.8.3
    docker push gcr.io/nx-live-platform-handle-w-care/kibana:6.8.3

La operación aquí es básicamente descargar la imagen del repositorio original,
reetiquetarla para nuestro repositorio y cargarla en el mismo. Por supuesto,
para que todo esto funcione correctamente es necesario que los permisos de
acceso a nuestro repositorio desde Docker estén correctamente configurados.

#Despliegue de los charts
Con todo preparado, es hora de realizar el despliegue de los charts. Lo podemos
hacer mediante las siguientes órdenes:

    helm install --name elasticsearch elasticsearch --version 6.5.0 --set imageTag=6.8.3 --set replicas=6 --set service.type=LoadBalancer --set image=gcr.io/nx-live-platform-handle-w-care/elasticsearch --set volumeClaimTemplate.resources.requests.storage=1Ti
     helm install --name kibana kibana --set imageTag=6.8.3 --set ingress.enabled=false --set service.type=LoadBalancer --set image=gcr.io/nx-live-platform-handle-w-care/kibana

Una vez hecho esto, los pods deberían iniciar correctamente y, después de
algunos instantes, los servicios deberían recibir IP externas a través de las
cuales acceder, tanto a kibana como a elasticsearch.
