+++
categories = ["CI/CD","Jenkins","Maven"]
date = "2019-09-01"
title = "Jenkins: Construcción de una Jerarquía de proyectos Maven"
type = "post"
draft = false
+++
Demostración de la utilización de las tareas tradicionales de Jenkins para
la construcción de una jerarquía de proyectos dependientes entre sí.

En este post vamos a estudiar las capacidades de *Jenkins* para:

- Gestionar herramientas automáticamente de forma global.
- Manejar tareas Maven de forma sencilla con el plugin Maven Integration.
- Reconocer y gestionar automáticamente las dependencias entre proyectos Maven.
- Manejar distintos ámbitos para los repositorios locales Maven.

# La jerarquía de proyectos

Tenemos a trabajar con 3 repositorios: *blog-base*, *blog-medio* y *blog-final*.
Están alojados de forma pública en [GitHub](https://www.github.com).
Sus respecticas url son las siguientes:

- https://github.com/jorgealmagro/blog-base.git
- https://github.com/jorgealmagro/blog-medio.git
- https://github.com/jorgealmagro/blog-final.git

Cada uno de estos repositorios contiene un proyecto Maven. Inspeccionando los
archivos *pom.xml* de cada uno de estos proyectos vemos que existe una jerarquía
de dependencias entre ellos, resumida en la siguiente imagen:

{{< figure src="jerarquia.svg" title="La jerarquía de proyectos" >}}

El proyecto *blog-medio* depende del proyecto *blog-base*. El proyecto
*blog-final* depende de los otros dos.

## Compilación y ejecución manual del proyecto

El proyecto *blog-final* contiene un manifiesto que permite la ejecución del
JAR generado.

La siguiente secuencia de comandos, partiendo de un directorio vacío, descarga
todos los repositorios y compila los proyectos para generar el JAR final:

{{< highlight bash >}}
git clone https://github.com/jorgealmagro/blog-base.git
git clone https://github.com/jorgealmagro/blog-medio.git
git clone https://github.com/jorgealmagro/blog-final.git
mvn -f blog-base install
mvn -f blog-medio install
mvn -f blog-final install
{{< /highlight >}}

Al finalizar, en el directorio *blog-final/target* encontraremos el
artefacto *blog-final-0.0.1-SNAPSHOT.jar* que podemos ejecutar mediante:

{{< highlight bash >}}
java -jar blog-final/target/blog-final-0.0.1-SNAPSHOT.jar
{{< /highlight >}}

Esto debería generar la siguiente salida por consola:

{{< highlight bash >}}
Inicio de Final.main()
Creado objeto Base
Creado objeto Medio
Creado objeto Base
Final.main() terminado
{{< /highlight >}}

# Creación de la primera tarea

Vamos a partir de una instalación nueva de Jenkins. En la [documentación de
Jenkins](https://jenkins.io/doc/book/installing/) podemos encontrar muchas
opciones para crearla.

Podemos utilizar cualquiera que esté disponible para el sistema en el que
se vaya a ejecutar. En mi caso, prefiero la utilización de un contenedor Docker
porque una vez terminado se puede eliminar, quedando el sistema base totalmente
limpio.

Inicializamos el sistema con la configuración de plugins por defecto y creamos
el usuario administrador que utilizaremos para el resto del artículo.

Una vez instalado e inicializado, lo primero que haremos es crear las tareas
Jenkins con la configuración necesaria para gestionar cada uno de los
repositorios que preparamos anteriormente.

Desde la página inicial, seleccionamos la opción `Nueva tarea`. Jenkins nos
muestra los tipos de tarea disponibles. Observamos que la tarea que necesitamos
`Proyecto Maven` no se encuentra en la lista. Esto es debido a que el plugin
que la define no está entre los que se instalan por defecto durante la
inicialización.

{{< figure src="jenkins-default-task-types.png" title="Tipos de tarea por defecto" >}}

Para instalar el plugin volvemos a la página principal, seleccionamos la
opción `Administrar Jenkins` y `Administrar Plugins`. Vamos a la pestaña
`Todos los plugins` y, con la ayuda del filtro, buscamos el plugin `Maven
Integration` y lo instalamos. Debemos prestar atención aquí y no confundirnos
con el plugin `Pipeline Maven Integration`, que proporciona funcionalidades
de integración con Maven para los trabajos de tipo Pipeline.

Una vez instalado, reintentamos la creación de una nueva tarea. Ahora sí,
entre las opciones disponibles se encontrará `Crear un proyecto maven`:

{{< figure src="jenkins-maven-task-type.png" >}}

La primera tarea que vamos a crear es la que gestionará la construcción del
repositorio *blog-base*. Asignamos *blog-base* como nombre de tarea,
seleccionamos `Crear un proyecto maven` y pulsamos `Ok`. Esto creará la tarea
y nos llevará a la página de configuración.

En la página de configuración es dónde se definen todos los parámetros
necesarios para la ejecución de la tarea. Uno de estos parámetros es la
configuración de herramienta Maven que vamos a utilizar, entre todas las
definidas en Jenkins.

### La definición de herramientas en Jenkins

Jenkins permite la definición de un conjunto de herramientas que son instaladas
automáticamente. Esto simplifica la configuración de los sistemas,
tanto la máquina dónde se ejecuta el servidor maestro como los ejecutores
esclavos que puedan utilizarse para la distribución de tareas, al evitar tener
que instalarlas manualmente.

Una característica de las herramientas gestionadas automáticamente por Jenkins
es que permiten la definición de múltiples configuraciones independientes.
Las tareas, cuando requieren una de estas herramientas, pueden especificar
la configuración que quieren utilizar entre todas las disponibles. De esta
forma se permite que distintas tareas puedan utilizar diferentes
versiones, simplificando la creación de tareas que necesitan versiones de las
herramientas incompatibles entre sí. En otro caso, si la herramienta tuviese que
ser global al sistema, haría muy difícil la ejecución de tareas incompatibles en
el mismo sistema, obligando a utilizar distintos sistemas para soportar las
distintas configuraciones o, peor aún, a gestionar la configuración de la
herramienta desde la propia tarea.

La definición de las configuraciones de las herramientas en Jenkins se realiza
desde la opción `Global Tool Configuration` en la zona de administración.

Jenkins es capaz de gestionar la instalación de las herramientas más habituales,
como Git, JDK, Gradle, entre otras. Para definir una instalación de Maven,
debemos ir a la opción correspondiente y pulsar el botón `Añadir Maven`:

{{< figure src="jenkins-maven-add-installation.png" >}}

Existen múltiples opciones para la definición de las herramientas Maven en
Jenkins. Entre las opciones disponibles están las de utilizar herramientas
ya disponibles en el sistema o, más interesante, realizar la instalación
automática de una determinada versión.

Para utilizar una versión disponible en el sistema simplemente deberemos indicar
el directorio dónde se encuentra instalada en el parámetro **MAVEN_HOME**. El
mayor inconveniente de esta opción es que para que funcione correctamente
durante la distribución de trabajos a ejecutores esclavos, debemos tener la
herramienta instalada en todas las instancias que utilicemos exactamente en la
misma ubicación.

La otra opción, dónde realmente se aprecia la capacidad de Jenkins para la
gestión de las herramientas, es la instloungealación automática. Aquí nos
encontraremos con 4 opciones:

- **Instalar desde Apache**: La más sencilla de todas. Especificamos una versión
  entre todas las disponibles y Jenkins automáticamente la descarga desde los
  servidores de Apache y la instala en los sistemas.
- **Ejecutar comando**: En este caso se especifica un script que será
  ejecutado para realizar la instalación de la herramienta.
- **Ejecutar Batch**: Similar al caso anterior pero para el caso en que la
  máquina sea Windows
- **Extraer \*.zip/\*.tar.gz**: Se indica una URL desde la que se puede
  descargar un archivo comprimido que será extraído en un directorio especifico
  del sistema.

Para nuestro caso es suficiente con la instalación básica. Definimos un nombre
para la instalación, por ejemplo *M3_TEST* y configuramos un instalador
automático desde Apache para la última versión disponible, la *3.6.2* en el
momento de escribir esto.

### Finalizando la creación de las tareas

Una vez hemos creado la instalación de la herramienta Maven, podemos continuar
con la configuración de la tarea. Para ello, volvemos a la página principal
y veremos que aparece la tarea *blog-base* que creamos anteriormente. La
seleccionamos y pulsamos en la opción `Configurar` para poder retomar
la configuración dónde la dejamos anteriormente.

Si ahora vamos a la sección **Proyecto** vemos que ha desaparecido el mensaje de
error que indicaba que debíamos definir una configuración de la herramienta.

Como sólo hemos definido una configuración, Jenkins la utiliza automáticamente
sin preguntar. En caso de que existiese más de una configuración, se nos
permitiría especificar aquí cuál es la que queremos utilizar para esta tarea.

Lo primero que debemos definir es el repositorio desde el que la tarea debe
obtener el código fuente. En la sección **Configurar el origen del código
fuente** se proporcionan mútiples opciones, dependiendo de los plugin que estén
instalados en el sistema. Seleccionamos la opción **Git** y especificamos la
URL del repositorio donde realizamos el fork de *blog-base*.

No es necesario especificar ninguna credencial ya que el repositorio es
público. El resto de los parámetros los dejamos con los valores por defecto.
De momento sólo vamos a recostruir los cambios en la rama master del
respositorio.

Una vez hemos indicado desde dónde se debe obtener el código fuente lo siguiente
es indicar qué es lo que haremos con él. Bajamos hasta la sección **Proyecto**
y en el campo **Goals and options** indicamos la meta Maven a ejecutar, en
nuestro caso *install*.

Hecho esto ya tenemos definida la configuración básica. Guardamos y volvemos
a la página principal de la tarea.

## Comprobando la tarea

Vamos a verificar que la tarea funciona correctamente. Desde la página principal
de la tarea seleccionamos la opción `Construir ahora`. Después de unos instantes
veremos aparecer en la historia de tareas una entrada indicando que existe
una ejecución en progreso:

{{< figure src="jenkins-task-history.png" >}}

Pulsamos en la bolita a la izquierda del número de tarea y veremos el log de
la salida por consola de la ejecución de la tarea. Si todo es correcto,
deberíamos ver el mensaje **BUILD SUCCESS** al final del mismo.

Si revisamos el diagrama de dependencias al comienzo del artículo vemos que
el repositorio base no tenía ninguna. A continuación vamos a continuar creando
la tarea Jenkins para la construcción del repositorio *blog-final*. Si
recordamos, este repositorio depende tanto de *blog-base* como de *blog-medio*.
Como aún no hemos realizado la compilación de este último, es lógico esperar
que la ejecución de la tarea de *blog-final* falle.

## Creación de una tarea con dependencias no resueltas

Repetimos los pasos para la creación de la tarea Maven que permita la
compilación del código en el repositorio *blog-final*:

- En la página inicial seleccionamos `Nueva Tarea`.
- Le damos como nombre *blog-final* y la creamos de tipo `Crear un proyecto
  maven`.
- Configuramos el repositorio de código fuente y la meta *install* de la
  misma forma que hicimos para *blog-base*.

Una vez creada, ejecutamos la tarea pulsando `Construir ahora`. Si vamos a la
consola de ejecución veremos que en este caso el proceso falla con un mensaje
indicando que la dependencia blog-medio no ha podido resolverse:

{{< highlight bash >}}
[ERROR] Failed to execute goal on project blog-final: Could not resolve
dependencies for project es.almagrofernandez:blog-final:jar:0.0.1-SNAPSHOT:
Could not find artifact es.almagrofernandez:blog-medio:jar:0.0.1-SNAPSHOT ->
[Help 1]
{{< /highlight >}}

## Creación de la tarea para el repositorio intermedio

El fallo de compilación anterior se debe a que maven no puede encontrar la
dependencia *blog-medio* que necesita *blog-final* para compilar, porque aún
no ha sido instalada en el repositorio local de la máquina Jenkins.

Para resolver este problema vamos a crear la tarea que compila el repositorio
que falta.

Esta vez en lugar de crearla desde cero vamos a copiar una de las tareas
existentes. Para ello volvemos a seleccionar `Nueva Tarea` en la página
principal y vamos a nombrar la nueva tarea *blog-medio*, pero en lugar de
seleccionar un proyecto Maven vamos a ir a la última de las opciones y en
el campo `Copy from` vamos a introducir el nombre de una de las dos tareas
que ya tenemos, por ejemplo *blog-base*. Pulsamos `Ok`.

Hecho esto, llegaremos a la misma página de configuración de una tarea Maven
pero esta vez, en lugar de estar vacía vemos que viene preinicializada con
los valores de la tarea que hemos copiado.

El único cambio que tenemos que hacer es en la URL del repositorio de código
fuente, que deberemos modificar para que apunte al repositorio donde hemos
realizado el fork de *blog-medio*.

Guardamos la configuración de la tarea y volvemos a la página principal.

## Compilación final de todos los repositorios

Ya tenemos todas las tareas necesarias para la gestión de nuestra base de
código.

Para finalizar la compilación con éxito, empezaremos ejecutando la tarea
*blog-medio*, lo que instalará el artefacto que necesitaba *blog-final* para
poder ser compilado.

Si nos damos cuenta, una vez termina la ejecución de la tarea *blog-medio*,
Jenkins inicia automáticamente la ejecución de la tarea *blog-final*. Esto se
debe a una característica muy importante de las tareas Maven: la detección
automática de dependencias.

Cada vez que se ejecuta una tarea Maven, Jenkins toma nota tanto de los
artefactos que son generados como de los que depende esa tarea. Con esta
información, construye el grafo de dependencias automáticamente de forma que
cuando detecta que se ha generado una nueva versión de un artefacto determinado,
encadena ejecuciones de las tareas que dependen de dicho artefacto de forma que
se comprueba que las modificaciones al código de un repositorio no rompen el
código de los repositorios dependientes.

Por supuesto esto se realiza tantas veces como sea necesario hasta que todas
las tareas dependientes han sido ejecutadas.

Podemos comprobar esto último si lanzamos una ejecución manual de *blog-base*.
Una vez ha finalizado la ejecución, comenzará una ejecución de la tarea
*blog-medio* y, una vez terminada, comenzará por último una ejecución de la
tarea *blog-final*.

Es importante darse cuenta de que Jenkins es lo suficientemente inteligente como
para detectar que, aunque *blog-final* depende de *blog-base* directamente,
no lanza la ejecución inmediatamente al finalizar esta última sino que, como
también hay una dependencia intermedia *blog-medio* que debe ser así mismo
construida, espera a que ésta termine para finalmente lanzar la tarea
*blog-final*.

## Bloqueo de la ejecución automática de una tarea dependiente

Si por cualquier motivo queremos impedir la ejecución automática de una tarea
cuando se contruye una nueva versión de alguno de los artefactos de los que
depende, podemos hacerlo yendo a la tarea a bloquear y deseleccionando la
opción **Ejecutar siempre que cualquier 'SNAPSHOT' de los que dependa sea
creado**.

Podemos probarlo yendo a la tarea *blog-medio* y deseleccionando dicha opción.
Una vez hecho, lanzamos una ejecución de *blog-base*. Veremos que una vez
finalizada la ejecución de esta, se encadenará una ejecución de la tarea
*blog-final*, saltándose por tanto la ejecución de *blog-medio* debido al
cambio de configuración realizado.

# Ámbitos del repositorio local Maven

Por último vamos a revisar las capacidades de Jenkins para la gestión del
repositorio local Maven.

En nuestro caso, hemos visto que es necesario cierto paso de información entre
las 3 tareas. En concreto, para que las tareas dependientes *blog-medio* y
*blog-final* puedan ser ejecutadas correctamente, es necesario que los
artefactos generados por las tareas de las que dependen estén disponibles.

Si observamos el archivo **pom.xml** de alguno de estos dos proyectos, veremos
que no hay ninguna información relativa a la forma en la que se deben obtener
dichas dependencias. Entonces ¿cómo hace Maven para poder pasar las dependencias
entre proyectos? La respuesta se encuentra en el *repositorio local*.

Cada vez que se realiza una ejecución de la meta *install*, lo que se hace,
entre otras cosas, es almacenar una copia del artefacto, junto con algo de
información adicional, en una ubicación local, que por defecto está en
el directorio .m2 del directorio raíz del usuario.

Cuando un proyecto Maven necesita resolver una dependencia, se mira dentro
de dicha ubicación local y, si se encuentra, se da la dependencia por resuelta.

De esta forma, cuando ejecutamos la tarea *blog-base*, lo que hacemos es
instalar una versión del artefacto generado en el repositorio local que puede
posteriormente ser utilizado por las otras dos tareas *blog-medio* y
*blog-final*.

Este es el comportamiento por defecto de Maven y también es el que por defecto
utiliza Jenkins para la ejecución de las tareas Maven. Jenkins proporciona otras
dos opciones para la gestión del repositorio local: *local al ejecutor* y
*local al espacio de trabajo*.

Un repositorio *local al ejecutor* creará dentro del nodo tantos respositorios
como ejecutores tenga activados. Dos tareas Maven que ejecuten en el mismo
ejecutor compartirán los artefactos archivados en dicho repositorio.

Un repositorio *local al espacio de trabajo* creará un repositorio para cada
espacio de trabajo. Sólo las tareas que ejecuten en ese espacio de trabajo
serán capaces de ver los artefactos almacenados en el mismo.

Jenkins crea por defecto un espacio de trabajo independiente para cada tarea.
Nuestras 3 tareas, por tanto, tienen espacios de trabajo independientes. Si
cambiamos el tipo de repositorio que utilizan a *local al espacio de trabajo*
podemos observar como se bloquea el intercambio de artefactos entre tareas,
haciendo que las ejecuciones fallen.

Podemos verificar este concepto con el siguiente procedimiento:
- Para cada una de nuestras 3 tareas, en el área de Proyecto pulsamos en el
  botón **Avanzado...** y en la lista de opciones que aparece marcamos
  **Utilizar un repositorio Maven privado**.
- En el campo **Estrategia** que se muestra a continuación, seleccionamos la
  opción *Local to the workspace*

Una vez hecho para cada una de las 3 tareas, volvemos a ejecutar manualmente la
tarea *blog-base*. Esta ejecución finalizará correctamente y, tal y como se
espera, se lanzará la ejecución de la tarea dependeinte *blog-medio*. Esta tarea
fallará con el siguiente error:

{{< highlight bash >}}
[ERROR] Failed to execute goal on project blog-medio: Could not resolve
dependencies for project es.almagrofernandez:blog-medio:jar:0.0.1-SNAPSHOT:
Could not find artifact es.almagrofernandez:blog-base:jar:0.0.1-SNAPSHOT
-> [Help 1]
{{< /highlight >}}

Este mensaje indica que Maven no ha sido capaz de resolver la dependencia que
tiene del artefacto generado por *blog-base*, ya que se encuentra en un
repositorio local a otro espacio de trabajo.

# La comunicación de artefactos entre tareas y la ejecución multinodo
La comunicación de artefactos entre tareas es un requisito muy importante que es
satisfecho sólo parcialmente mediante la utilización de repositorios locales
comunes entre tareas.

En el caso de utilizar la estrategia por defecto de tener un único
repositorio local común a todos, esto no resuelve el problema de la ejecución
de tareas en múltiples nodos separados.

Jenkins permite la distribución de las tareas en varios nodos independientes.
Estos nodos no comparten repositorios locales de artefactos, lo que impide
la comunicación de artefactos entre tareas que se ejecutan en nodos separados.

Para solucionar este problema, se deben utilizar servidores de gestión de
repositorios como Nexus o Artifactory (JFrog), que permiten el almacenamiento
de artefactos de forma que pueden ser compartidos entre tareas Maven con
independencia de dónde se ejecuten (siempre que tengan acceso al repositorio).

En otro artículo, estudiaremos la forma de instalar un servidor Nexus
y cómo configurar Maven y Jenkins para que hagan uso de él.
