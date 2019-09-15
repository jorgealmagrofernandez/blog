---
title: "Instalación y uso de Nexus"
date: 2019-09-10T20:02:30+02:00
draft: true
---

Configuramos el settings.xml para autorizar el acceso al Nexus:
  Instalamos el plugin de gestión de ficheros de configuración
  Creamos un fichero de settings.xml para maven con los parámetros adecuados
  Tenemos que crear unas credenciales para poder incluirlas en la propia
  sección server del fichero de forma automática
  Configuramos el fichero settings global en la zona de configuración global
  de herramientas
  Volvemos a lanzar la ejecución y comprobamos que falla (parece que la
  configuración global no funciona correctamente)
  Hacemos el cambio en el job y parece que esta vez resuelve correctamente
  las dependencias a través del repositorio de artefactos

Configuramos la consulta al repositorio como disparador de consultas para
  determinar cuándo debe lanzarse una nueva ejecución.
