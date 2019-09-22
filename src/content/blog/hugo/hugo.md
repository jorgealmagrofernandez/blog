+++
categories = ["Hugo"]
date = "2019-01-01"
title = "Cositas acerca de Hugo"
type = "post"
draft = true
+++

Instalación de Hugo en Manjaro:
> sudo pacman -S hugo

Comenzar un nuevo sitio:
> hugo new site <dir>

Configuración de un tema:
> git clone o submodule dentro de la carpeta themes del tema a utilizar
> Añadir la línea theme = "<nombre-del-tema>" al archivo config.toml

Crear un nuevo contenido:
> hugo new <categoría>/<archivo>.<formato>

Iniciar el servidor:
> hugo server -D
