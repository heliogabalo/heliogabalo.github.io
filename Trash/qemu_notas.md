1. [PROCESO DE INSTALACION DE UNA IMAGEN](#i1) 
    1. [Crear imagen](#i1)  
    2. [Convertir imagen](#1i2)
    3. [Redimensionar imagen](#1i3)

2. [TRABAJAR CON UNA COPIA DE IMAGEN](#2i)
    1. [Backing-files/overlays](#2i)
    2. [Snapshots](#2i2)  
        - [Captura interna](#2i2a)  
        - [Captura externa](#2i2b)  
        - [Estado de la VM](#2i2c)
    3. [Creando capturas](#2i3)  
    4. Proceso de reversión  
    5. Confluencia en las capturas  
    6. Aprovado de bloque  
    7. Aceptación o emisión de bloque  
    8. Flujo de línea?  
    9. Borrado de capturas  
    10. Notas de autor 
3. [CON O SIN CONEXION A INTERNET](#3i)
    1. [Modo usuario](#3i1)  
       - [Configurar una MAC específica](#3i1a)  
    2. Modo Tap
4. [EL LOOPBACK](#4i)
    1. [MONTAR UN LOOPBACK PARA COMUNICARNOS CON LA VM SIN CONEXION](#4i1)  
    2. [LOOPBACK PARA UNA IMGEN (USANDO MODULOS EN EL KERNEL)](#4i2)  
      - Lanzar la VM apuntando al servidor NBD
5. EXPERIMENTAL
6. [ATAJOS DEL TECLADO](#6i) 
    1. [Comandos del monitor qemu](#6i1) 
    2. [Redefinir teclas](#6i2)
7. [AGRADECIMIENTOS](#ai)

---
## 1. <a name="i1">PROCESO DE INSTALACION DE UNA IMAGEN</a>  

#### Crear imagen

Bien sea porque tenemos el disco original (en este caso un SO windows)  
o bien porque lo hayamos descargado, deberemos antes CREAR una imagen GUEST  
con la que  QEMU, pueda trabajar.
  
Para esto primero creamos la imagen. Una "caja" vacía:

    qemu-img create -f qcow2 mi_imagen.img 1G

Aquí el flag "-f" indica el tipo de formato con el que será creada nuestra imagen (aún vacia).  
También indicamos el archivo imagen y el tamaño en Gigabytes.

Instalación de SUPUESTO OS en la imágen previamente creada:
 
    qemu -m 256 -hda mi_imagen.img -cdrom winxpsp2.iso -boot d  
  
Este comando anterior es un poco confuso.
- Habrá que sustituir "qemu" con el comando apropiado, en relación a la arquitectura  
del sistema operativo GUEST con el que se vaya a trabajar. En este caso sería:  
_qemu-system-i386_
- Nuevamente el flag -m indica la memoria RAM para el SUPUESTO SO.
- La siguiente opción -hda indica el archivo imagen donde vamos a instalar la imagen  
del SO.
- Sigue la opción -cdrom. Parece indicar el dispositivo físico un 'CD', pero todo  
apunta a que se trata de una denominación para diferenciarlo de la partición GUEST  
que acabamos de crear. Es decir, que utilizaremos el mismo flag '-cdrom' para tratar  
con una imagen descargada en el disco duro, o una imagen que previamente hayamos  
'quemado' en un 'CD'.  
La diferencia es que para utilizar un 'CD' a la hora de hacer la instalación en  
nuestra 'caja vacía' habrá que indicar la ruta hacia el dispositivo ejem. /dev/cdrom  
La opción -boot d indica como 'cadena' la letra que será usada en el arranque del sistema.  
Es exactamente igual a como interpreta la BIOS el 'orden' de arranque de sistema de  
nuestro HOST.  
    - 'a' y 'b' para la floppy
    - 'c' para el disco duro
    - 'd' para el CD-ROM
    - 'n-p' arranque desde RED. Opcion muy interesente para un GUEST. Investigar!!!  
Desde Linux, la cadena que representa el dispositivo de arranque, está muy claro,  
(pues nosotros no usamos letras para esto). Así que 'c' claramente representa al  
disco duro y 'd' a un CD-ROM.  
Desde una perspectiva Windows, habrá que asegurarse. Pués windows utiliza letras para  
denominar los dispositivos de almacenamiento.

#### <a name="1i2">Convertir imagen</a> 

Por qué convertir imágenes antes de instalarlas:  
    
  # qemu-img convert -f vhd -O qcow2 source.vhd destination.qcow2
    
> Es posible que la instrucción no funcione correctamente debido a algún
> cambio en la version utilizada con _qemu_. Este otro comando debería funcionar.

    qemu-img convert -O qcow2 filename file_output  (autodetectada??)


Qemu tiene el conversor de imagenes mas versatil, en relación a otros emuladores.  
Esto lo convierte en una herramienta indispensable a la hora de trabajar con VMs.  
Así como otros emuladores, presentan limitaciones a la hora de trabajar con imagenes  
específicas, qemu es capaz de interpretar una gran variedad de éstas, además de  
poseer un tipo genérico 'raw' donde converge con otras 'versiones'.

---
## <a name="1i3">Redimensionar imagen</a>






## <a name="2i">TRABAJAR CON UNA COPIA DE IMAGEN</a>  

#### Backing_files/overlays

La principal idea aquí, es la 'copia de seguridad'. Una vez se ha instalado el sistema  
operativo, puede trabajarse sobre un archivo de 'prueba/efecto'. Al que llamamos  
_Overlay_.  
Esto permite probar extensivamente un determinado GUEST, sin importar los cambios que  
hagamos, pues no serán aplicados al GUEST original, sino a la copia.  
La imagen del archivo que contiene la instalación original, o en un estado básico,  
la llamamos _BackingFile_.  

Para preparar este _entorno de prueba_, primero se crea una imagen en crudo, asignando un  
tamaño a la misma.

    $ qemu-img create -f raw image_file.raw 10G

A continuación creamos el backing_file. Realmente no lo estamos creando, estamos formando  
la imágen en crudo, para que reconozca nuestro entorno de prueba, asociando ambos  
archivos: _raw/qcow2_ en este caso.  
Lo hacemos con la siguiente línea:

    $ qemu-img create -o backing_file=image_file.raw,backing_fmt=raw \  
      -f qcow2 overlay.cow   

Lo mas importante en este proceso, es asegurarnos de que el 'overlay' apunta al  
backing_file. Podemos comprobarlo con la aplicación _file_

    $ file overlay.cow  

> Tip: Cuando trabajamos con procedimientos de este tipo, es habitual separar los  
> archivos, en distintos directorios. Una forma sencilla y eficaz de hacerlo  
> sin tener que estar escribiendo una y otra vez rutas largas, es asignar  
> la ruta a una variable ejem: crear_backing.sh

    #!/bin/sh  
    
    my_path=/ruta/a/directorio/respaldo  

Se que algunos me tacharán de novato, pero escribiendo las rutas directamente en la línea  
de comando, no conseguí de ninguna manera, que el vínculo entre ambos: backing-overlay,  
no se rompiese.  

> CAZADO:
> Al llamar al 'backing_file' en el proceso de instalación de la imagen, qemu, parece  
> no reconocer direcciones fuera del directorio que contiene la imagen 'base'. Esto  
> quiere decir que para instalar la imagen en el backing file es necesario encontrarse  
> en el directorio contenedor: mezcla las rutas absolutas/relativas.  



La VM arranca con:  
  
    qemu overlay.cow -m 128 




#### <a name="2i2">SnapShots</a>

Snapshot es la captura de _estado_ de una determinada máquina virtual, en un momento  
concreto. Esto incluye al sistema operativo y todas las aplicaciones. Es como  
una fotografía instantánea: _en ella aparecerá todo lo que hay delante del objetivo de  
la cámara_.

Aquí hablamos de instancias, como lo haríamos sobre una _clase_, por que en realidad  
es un concepto similar: definimos un _proceso_, que ya ha sido implementado en otro  
_escenario_. Configurándolo para cumplir una _tarea específica_.  

Esto evita tener que modificar el proceso original, y trabajar directamente en él, con  
todas, o muchas, de sus características.

La casualidad no existe. Qcow2(copy-on-write)podría traducirse como:  
"escritura sobre la copia", que es exáctamente lo que se pretende en este _proceso_. 

Esta ténica puede ser tan complicada o simple como la necesidad a cubrir, pero siempre  
guarda la misma idea: mantener a salvo el archivo original, y realizar cambios, sobre  
una _copia_.

Al realizar los cambios, modificaciones, pruebas, etc. aparece la alternativa de guardar  
ese _estado_ en la imagen origanl, o tal vez descartarlo, por que  ha sido un _horrible  
desastre_.

Empezamos crenado una relación BackingFile/Overlay:

    $ qemu-img create -b $mi_Ruta/base.img -f qcow2 \  
      $mi_ruta/Overlays/overlay1.qcow2  
    $ qemu.img create -o backing_file=$mi_ruta/base.img,backing_fmt=raw \  
      -f qcow2 $mi_ruta/Overlays/overlay2.qcow2  

El flag *-b*, parece referirse a la *base*, pero ha queado obsoleto desde la version  
_qemu_ actual. Es utilizado junto al comando _commit_ que será visto, mas adelante.

> __man page:__  

    commit [--object objectdef] [--image-opts] [-q] [-f fmt] [-t cache] [-b base] [-d] [-p]  

> La bandera(flag):  __-o__ significa opciones. Cuando la imagen de disco, es creada con la  
> _opción_ *backing_file*, la imagen(overlay), sólo guardará la diferencia respecto a la base.  
> El tamaño del archivo, puede ser omitido.  
> **-f** hace referencia al formato de archivo, para el __overlay__. Puesto que es habitual  
> guardar la _base_ en _crudo_.  
> **$mi_ruta:** no es más que una varible, que he utilizado para simplificar la línea.  

Es importante comprobar que el _vínculo_ entre ambos archivos, es el _adecuado:_  

    $ file _archivo_  
    $ qemu-img info --backing-chain $mi_ruta/Overlays/overlay2.qcow2  

__file__ ofrece una versión resumida si únicamente buscamos comprabar el vínculo.  
__qemu-img info --backing-chain__ aporta información más detallada:  
    
    image: /path/to/BF/Overlays/img1.cow
    file format: qcow2
    virtual size: 3.0G (3221225472 bytes)
    disk size: 1.5G
    cluster_size: 65536
    backing file: /path/to/BF/image_file.raw
    backing file format: raw
    Format specific information:
        compat: 1.1
        lazy refcounts: false
        refcount bits: 16
        corrupt: false
    
    image: /path/to/BF/image_file.raw
    file format: raw
    virtual size: 3.0G (3221225472 bytes)
    disk size: 1.4G

Otro usuario **Linux**, desde la distribución _Fedora_, ha querido incluir en su documentación,  
un conjunto de términos utilizados junto a estas _capturas de estado_. Intentaré  
traducirlos sin cambiar su contenido...  

#### <a name="2i2a">**Captura interna:**</a>  

Un archivo qcow2 que sostiene la captura y "delta" hasta el punto de guardado. Delta hace  
referencia al "direncial" escrito en la imagen, aquellas partes del disco que han sufrido  
modificación.

    $ ls -sh $my_path && ls -sh $my_path/Overlays
    1,5G image_file.raw  2,1G test01.img 
       0 Overlays           0 Unsafe     
    total 1,7G
    1,6G img1.cow  134M test_over.qcow2

>  En la última línea del siguiente bloque de código, puede verse como al archivo img1.cow,  
>  se han aplicando diferentes actualizaciones, quedando reflejadas en el tamaño de disco.  
>  El archivo test-over, represanta otra imagen, a la que se han aplicado "pocos" cambios.  


**Captura interna de disco:**  
El estado del disco virtual dado en un punto del tiempo. Tanto la captura, como _delta_  
son almacenados en el mismo archivo qcow2. Pueden ser tomados igualmente, cuando el  
SUPUESTO esta 'vivo/encendido' u 'offline/apagado'.  

  - libvirt: esta librería, usa el comando 'qemu-img' cuando el SUPUESTO está _apagado_.
  - libvirt: usa el comando 'savevm' cuando el SUPUESTO está _encendido_.

**Punto de guardado interno del sistema:**  

Estado de la _RAM_, estdo del dispositivo y el estado del disco de un SUPUESTO en carrera.  
Todos son guardados en el mismo archivo original qcow2. Puede ser tomado durante la  
carrera.  

  - libvirt: usa el comando 'savevm' cuando el SUPUESTO está _encendido_.  

> AVISO: savevm(captura en vivo), es un comando accedido a través del monitor qemu.  
> __ctrl+alt+2__ abre el monitor. El comando  _help_ lista la ayuda.  
> __ctrl+alt+1__ para volver al modo en el que hayamos lanzado la VM(gráfico/texto).  
> __q|quit__cierra qemu en modo monitor.  

    $ qemu-img info /path/to/img.qcow2
 
> Éste comando lista las capturas que hayamos tomado. Si no establecemos Id o tag  
> durante la captura con `(qemu )savevm id/tag`, es creado un nuevo archivo.

#### <a name="2i2b">**Captura externa:**</a>
Al tomar la captura se almacena el estado de disco en un archivo. En ese punto, la imagen  
se convierte a sólo lectura(_base_) y, un nuevo archivo(_overlay_) recogerá los _deltas_ del  
_estado_ guardado.  
 
**Captura externa de disco:**  
La captura de disco, es guardada en un archivo y, _delta_ hasta la captura, _seguido_ en uno  
nuevo, con formato qcow2. Puede ser tomada en _vivo_ o con la máquina apagada.  

  - libvirt: esta librería, usa el comando de shell 'transaction', durante la carrera del  
    SUPUESTO.  
  - libvirt: usa el comando de cónsola 'qemu-img' cuando el SUPUESTO está apagado.  
    

**Punto de guardado externo del sistema:**  
Aquí, el estado de disco del SUPUESTO será guardado en un archivo, su RAM y el estado  
del dispositivo serán almacenados en un nuevo archivo.

#### <a name="2i2c">**Estado de la VM**</a>
Guarda la RAM y el estado del dispositivo de un supuesto en carrera, sin embargo, no el  
estado de disco; a un archivo. Así, podrá ser restaurado más tarde.  
El proceso es similar a la hibernación de sistema.  
> _nota:_ el estado de disco, debería permanecer sin modificar, durante el tiempo de  
restauración.  

#### <a name="2i3">Creando capturas</a>  
Mediante el uso de una _captura externa_, una nueva imagen(**overlay**), es creada para  
facilitar la escritura del supuesto. La imagen previa se convierte en _captura_.


__Crear capturas internas de disco__
Dada la máquina `myVm`, es posible crear una captura con el siguiente comando de línea:  

    # virsh snapshot-create-as `myVm` capt1 `descripción-deCaptura`  

Funciona de la misma forma con o sin la _VM_ encendida. Se añade una breve descripción.  
Ahora sería oportuno listar y revisar los datos:  

    # virsh snapshot-list myVm  
    # qemu-img info /far/beyondThe/su/myVm.qcow2  

> _qemu-img info_ arroja información con detalle, sobre la captura interna.  


__Crear capturas externas de  disco__  
Primero es listado el dispositivo de bloque asociado a la supuesta.  

    # virsh domblklist myVm-base   <- domain block list  

A continuación es creada la captura, con la supuesta en carrera.  

    # virsh snapshot-create-as --domain myVm-base capt1 capt1-desc \
    --disk-only --diskspec vda,snapshot=external,file=/path/to/capt-de-myVm-base.qcow2 \
    --atomic  

La shell devuelve algo parecido a: `Domain snapshot capt1 created`  
Es entonces cuando la imagen de disco original myVm-base es convertida a un `backing_file`  


#### Borrado de capturas
Borrar __capturas internas__ sea en vivo o con la máquina apagada, no es complicado:  

    # virsh snapshot-delete --domain myVm --snapshotname snap6  

...alternativamente  

    # virsh snapshot-delete myVm snap6  

Mencionar aquí, que se está borrando la última captura, por lo que el vínculo con la  
_base_ no se rompe. Es de suponer, que si varias capturas han sido creadas, el orden  
en que borramos éstas, es importante. Alternativamente podemos corromper tranquilamente  
la imagen y pasar a otra cosa ...  


Libvirt aún no tiene la capacidad de borrar capturas externas, pero pueden llevarse a  
cabo con `qemu-img`.  

Supongamos; para no perder la costumbre, que se han tomado un par de capturas, sin  
aplicarse aún ningúna aceptación de cambio(commit):  

    $ qemu-img info /path/to/somewereIn/Overlays/test_over.qcow2
    image: /path/to/somewereIn/Overlays/test_over.qcow2
    file format: qcow2
    virtual size: 3.0G (3221225472 bytes)
    disk size: 808M
    cluster_size: 65536
    backing file: /path/to/image_file.raw
    Snapshot list:
    ID        TAG                 VM SIZE                DATE       VM CLOCK
    1         tagtag                 273M 2016-07-30 11:51:54   00:03:51.796
    2         idid                   273M 2016-07-30 11:52:10   00:04:01.967
    Format specific information:
        compat: 1.1
        lazy refcounts: false
        refcount bits: 16
        corrupt: false  

> La línea importante es la que dice `backing file`, hacia mitad de párrafo.  

Así que aquí no hay _problema_, podrían borarse ambas capturas, en cualquier orden.  
Pero son capturas internas y; sencillamente, no puden ser borradas. Fin de la historia.  
Sin la capacidad de usar virsh, es como cuando pica la oreja y uno se rasca la  
rodilla...


Borrar __capturas externas__  

Con la máquina apagada, de dos formas, puede realizarse la tarea:  
    `base <- capt1 <- capt2 <- capt3`  
>> La flecha se lee ...capt3 tiene su base en capt2 (capt==snapshot)  

  1. `base <- sn1 <- sn3`(copiando sn2 en sn1)  
  2. `base <- sn1 <- sn3`(copiando sn2 en sn3)

#### Metodo 1
El diagrama muestra la intención de hacer _desaparecer_ la captura 2, pero no sin antes  
_aceptar_ los cambios en alguna de las capturas contiguas.  
Igualmente, es necesario que sn1 no sea la _base_ de ninguna otra captura, de lo contrario  
tendríamos una base a la que han sido aplicados cambios, donde otras capturas esperan  
encontrarla sin esos cambios, consecuentemente los datos se malograían.  



---
> La implentación de la librería _libvirt_ está aún en desarrollo, por lo que parte  
> de su funcionalidad no se encuentra disponible, al menos en la rama estable. En  
> Debian los paquetes necesarios son libvirt-bin y libvirt-daemon, pero es probable  
> romper el sistema si se instalan desde ésta rama(alternativa _filo sangrante?_).  
> Actualizaré la sección, cuando el problema con estos paquetes sea resuelto.    
> Puede pasar mucho tiempo. 28-07-16.  
> Herramientas como _transaction_ _virsh_ no están disponibles.
> [Manual snapShots][fedora] -- en inglés.  

---  
## <a name="3i">CON O SIN CONEXION A INTERNET !!</a>

> __nota:__
>      Si se trabaja con el _overlay_, habrá que recordar actualizar el _backing_file_, de otra  
>      manera, estaremos escribiendo una y otra vez, las opciones necesarias para que la   
>      GUEST inicie a nuestro gusto. vínculo a trabajar con una copia de imagen  


Seguramente no seré la única persona que esté tratando de instalar un windows en una GUEST,  
a mi me dió algunos problemas con los dispositivos y drivers para la NIC. Que he solucionado  
añadiendo como apendice, la opción que detalla la línea de comando un poco más abajo.  

Da la casualidad que mi tarjeta de red es exactamente la que instala qemu, una Realtek,  
así que desconozco si a otras personas les será esto de ayuda.  

Pero para que conste el dato: cuando instalamos el sistema operativo en la guest, no hay  
que indicar ningún comando especial para tener acceso a internet. Por defecto, qemu  
establece el modo usuario.  

Mas bien el problema intuyo que viene dado desde el GUEST, cacawin(windrop in english)  
Es decir, que si estais buscando desde:  
     >>Panel de contro>>herramientas administrativas>>computer managament>>device manager  
     >>network adapter(dispositivo)  
La forma de instalar un dispositivo y su draiver, mejor quitaoslo de la cabeza.  
Porque hay chorrocientas alternativas y hay que pensar que qemu está instalando un dispositivo  
'virtual'.  

El dispositivo que ha quedado instalado en mi GUEST:  
  _Realtec RTL8139 family PCI Fast ethernet NIC_ (y su draiver con nombre similar/igual).  

En caso de que este dispositivo no funcione lo mejor es echar mano del manual de  
qemu-system-tu-_arquitectura-de-maquina_ y mirar que alternativas hay:  
>    -net nic[,vlan=n][,macaddr=mac][,model=type] [,name=name][,addr=addr][,vectors=v]  
>        Create a new Network Interface Card and connect it to VLAN n (n = 0 is the default). The  
>        NIC is an e1000 by default on the PC target. Optionally, the MAC address can be changed  
>        to mac, the device address set to addr (PCI cards only), and a name can be assigned for  
>        use in monitor commands.  Optionally, for PCI cards, you can specify the number v of  
>        MSI-X vectors that the card should have; this option currently only affects virtio  
>        cards; set v = 0 to disable MSI-X. If no -net option is specified, a single NIC is  
>        created.  QEMU can emulate several different models of network card.  Valid values for  
>        type are "virtio", "i82551", "i82557b", "i82559er", "ne2k_pci", "ne2k_isa", "pcnet",  
>        "rtl8139", "e1000", "smc91c111", "lance" and "mcf_fec".  Not all devices are supported  
>        on all targets.  Use "-net nic,model=help" for a list of available devices for your  
>        target.  


>> Solo he podido probar esta teoría en mis máquinas!   

Añadiendo la opcion -net parametro _nic_, qemu instala una targeta virtual de red genérica.  
De hecho, el manual que sugiere gentoo, es el que me ha funcionado a mi. La versión de qemu  
que corre mi máquina es la 2.6, es posible que en sucesivas actualizaciones vemamos otros  
cambios.  

El comando quedaría algo así:  

    $ qemu-system-(arch) -net nic,model8139 ...   

Hay dos formas básicas de dotar a la VM con conexión a internet:

  - Modo usuario (slirp)
  - Modo Tap

#### <a name="3i1">Modo usuario:</a>  

Un problema con el que nos encontraremos, es que la tarjeta virtual que estamos creando  
tiene asociado otro compenente, una especie de CTR o conector que debe ser único para  
cada GEST.

Esto puede resolverse asociando el dispositivo al conector, mediante un ID único.  
Habrá que constituir una nueva interfase de red, sobre la que se realizarán las  
conexiones, tanto del Host como del Guest, una _Vlan_.  

_vinculo a VLAN ._  

> _nota:_  
>       ...Los administradores de red configuran las VLAN mediante software en lugar  
>       de hardware, lo que las hace extremadamente fuertes.  
>>> Wikia - Vlan.

    $ qemu-<arch> -net nic,vlan=id -net user,vlan=id  

> Cada uno de los dispositivos de red debe asociarse a su conector único!!  
> -net nic,vlan=id1 -net user,vlan=id2 __NO FUNCIONARÁ__  

> PROBLEMA: despúes de hacer la instalación via CTR/interface, sigue siendo  
> un requisito lanzar la app con tal asociación. De otro modo, el dispositivo  
> sigue instalado, pero la Vm no tiene acceso a la interface virtual(vlan).  


#### <a name="3i1a">Configurar una MAC específica</a>  

    -netdev user,id=mynet0,net=192.168.76.0/24,dhcpstart=192.168.76.9  


De esta forma la MAC de la VM tendrá un identificador por defecto.  
Esto puede ser un inconveniente, si corremos mas de una máquina, y queremos tener acceso  
a internet en todas ellas, puesto que la aplicación genera siempre la misma MAC.  

    $ qemu-system-i386 -net nic,macaddr=52:54:XX:XX:XX:XX -net vde disk_image  


Para que esto no ocurra debe indicarse un identificador. Reempaza las "X" con números  
hexadecimales arbitrarios, pero recuerda conservar las primeras dos cifras, que hacen  
referencia al _id_ de fabricante(qemu).  


> _Notas:_ 
>

>   Otra idea es probar qemu-ga. Éste es un demonio que funciona desde dentro de la  
>   SUPUESTA, así que en teoría, el host via injection/algo puede gestionar la particion  
>   de la VM, pero habrá que averiguar que tipo de operaciones puede hacer GUEST-AGENT.  
>


#### Modo Tap
> to be continued...

---   
## <a name="4i">EL LOOPBACK</a>  

La traducción de loopback significa: bucle hacia atrás(ma o meno), o 'camino de  
regreso'. La idea es utilizar una partición o disco, que se encuentra fuera de la  
imagen de la VM.  

Antes de seguir, debo recordar que las operaciones con particiones y sobre módulos  
que afectan directamente al _kernel_, hay que hacerlas con permisos de administrador.  
Por precaución, siempre es recomendable hacer este tipo de opereaciones con la  
VM apagada. De otra forma, se corre el riesgo de corromper los datos de la imagen.  

En determidas ocasiones, la máquina virtual no tiene conexión a internet. Es el caso  
de una instalación con Qemu. Así que la mejor forma de comunicarnos con la VM es  
mediante la técnica del _loopback_.

Las técnicas que describiré a continuación, pueden llevarse a cabo sobre particiones con  
formato de disco usease formateadas: ejem Ext3, NTFS, FAT32 etc. o sobre una imagén sin  
una partición en concreto.

Esto quiere decir, que puede montarse una copia de "respaldo" de _old-games_, en una imagen  
lo suficientemente grande, sin necesidad de crear una partición dentro de la imagen.  
En realidad lo que ocurre, es que la partición abarca todo el disco, no significa que no  
haya partición, significa que todo el disco está ocupado por una única partición.  
En este "marco" qemu no tiene problema para leer la tabla de nodos de partición, porque  
solo hay una, con un formato determinado!!

Así que antes de describir las dos alternativas, quiero dejar anotado un concepto que puede  
ser aplicado a ambas técnicas: _montaje simple_.

Si _no_ se crea una partición en la imagen(la VM), los datos podrán ser montados sobre  
cualquier formato. Sin embargo, si tenemos alojadas particiones dentro de la imagen,  
únicamente podremos montarlas en formato raw(crudo en inglés).  
Esto es para que qemu pueda manejar las particiones alojadas.  

En este caso como trabajaremos sobre una imagen ISO, parece apropiado seguir los pasos  
descritos al principio del artículo. Crear la caja vaía, y escribirla en el formato  
apropiado. Pero como problamente no queramos lanzar otra GUEST, sino únicamente acceder  
al contenido del la imagem. La operación de calcular el offset de la partición, puede  
ser omitida _-ver mas adelante_. 

Esto es de lo que hablaba: el _montaje simple_. Puede determinarse mirando el contenido de  
la imagen:

    $ file imagen.iso  
       imagen.iso: ISO 9660 CD-ROM filesystem data 'GRTMPVOL_EN' (bootable)  

Vemos que únicamente contiene una partición. A continuación, tan sólo queda montar la  
imagen. Para esto utilizamos la aplicación _losetup:_  

    $ losetup /dev/loop0 /path/to/imagen.iso

En caso de utilizar este método _montaje simple_, para evitar que el sistema nos devuelva  
algún mensaje de aviso, acerca de los permisos con los que se monta la unidad, podemos  
especificar que lo haga en modo solo lectura.

    # mount -o ro,loop /path/to/image.iso /mnt/point  


> __nota:__ aquí va otra nota sobre el uso de los shasum y file, sobre la importancia  
> de hacer las comprobaciones oportunas en cuanto a imágenes descargadas. Y un  
> especial comentario acerca del cambio que se produce en un sha, cuando queremos montar  
> una imagen con permisos de escritura. IMPORTANTE INVESTIGAR!  

#### <a name="4i1">MONTAR UN LOOPBACK PARA COMUNICARNOS CON LA VM SIN CONEXION</a>  
Este método es útil cuando necesitamos averiguar donde empieza la partición con la 
que vamos a trabajar. En el punto dos, puede verse en la última columna _System_
el tipo de partición que contiene la imagen de disco. La primera es una swap, la 
segunda debe ser una tipo EXT. 

Qemu-img no formatea la imagen de disco, crea una imagen de disco vacía, con una formato
de archivo, pero aún no tiene partición. Esto se ve claro cuando instalamos un sistema 
operativo dentro de la imagen creada con qemu-img. Es el propio sistema operativo que
vamos a instalar, quien crea la partición y le da formato. 

Solo quiero aclarar, que si creamos una imagen con qemu-img y, tratamos de copiar un
archivo dentro, NUNCA FUNCIONARÁ. Por que es como si antes de instalar el sistema operativo
en nuestro host, normal(no hablo de VMs), tratásemos de meter algo en el disco
duro, No chutaría nada, ni siquiera se encendería la pantalla.

Link aquí a fdisk dd gpart crear imagenes.

  Calcular el _offset_ antes de montar la imagen de disco.  
    
 1. Asociar el dispositivo de imagen de disco, a la partición que vayamos a montar.

    tux@venus:~> losetup /dev/loop0 /images/sles11sp1_base.raw  
    
 2. Tamaño de sector y número de inicio de sector, de la partición a montar.

    tux@venus:~> fdisk -lu /dev/loop0  
    
    Disk /dev/loop0: 4294 MB, 4294967296 bytes  
    255 heads, 63 sectors/track, 522 cylinders, total 8388608 sectors  
    Units = sectors of 1 * 512 = 512[1] bytes  
    Disk identifier: 0x000ceca8  
  
           Device Boot      Start         End      Blocks   Id  System  
    /dev/loop0p1              63     1542239      771088+  82  Linux swap  
    /dev/loop0p2   *     1542240[2]    8385929     3421845   83  Linux  
  
    [1] Tamaño del sector.

    [2] Sector de inicio de la partición.

 3. Calcular el _offset_ de inicio, de la partición.

    sector_size * sector_start = 512 * 1542240 = 789626880

 4. Borrar el loop y montar la partición, dentro de la imagen de disco. Con el cálculo del  
    offset dentro del directorio ya preparado.  
 
    tux@venus:~> losetup -d /dev/loop0  
    tux@venus:~> mount -o loop,offset=789626880 \  
    /images/sles11sp1_base.raw /mnt/sles11sp1/  
    tux@venus:~> ls -l /mnt/sles11sp1/  
    total 112  
    drwxr-xr-x   2 root root  4096 Nov 16 10:02 bin  
    drwxr-xr-x   3 root root  4096 Nov 16 10:27 boot  
    drwxr-xr-x   5 root root  4096 Nov 16 09:11 dev
    [...]  
    drwxrwxrwt  14 root root  4096 Nov 24 09:50 tmp  
    drwxr-xr-x  12 root root  4096 Nov 16 09:16 usr  
    drwxr-xr-x  15 root root  4096 Nov 16 09:22 var  

 5. Copiar uno o mas archivos dentro de la partición montada y desmontar al terminar.

    tux@venus:~> cp /etc/X11/xorg.conf /mnt/sles11sp1/root/tmp  
    tux@venus:~> ls -l /mnt/sles11sp1/root/tmp  
    tux@venus:~> umount /mnt/sles11sp1/  

 
#### <a name="4i2">LOOPBACK PARA UNA IMAGEN (USANDO MODUOS EN EL KERNEL)</a>  

Aquí primero preparamos el dispositivo que será leído por el módulo de kernel NBD.  
Para ello debemos tener cargado dicho módulo o cargarlo en todo caso. Recuerda que   
para llevar a cabo este tipo de operaciones, debemos escalar privilegios.

Primero comprobamos si tenemos instalados los módulos que vamos  
a usar  ... normalmente en /lib/... (editar)  
Comprobar si el módulo está cargado o no, en el sistema. Puede determinarse con:

    $ lsmod |cat -n |grep modulo-en-cuestion  

- La primera instrucción lista los módulos cargados en el kernel.
- Através de tubería cuantificamos su número, por razones de stress!!
- y le pasamos un filtro grep, para concretar la salida.

Si el modulo no está cargado, lo cargamos dándole un parámetro `max_part`  
para poder acceder a los nodos de cada una de las particiones.  
En caso de no iniciar la variable, como el valor por defecto es 0(cero)  
podrá accederse al disco, pero no a los nodos de ninguna de las particiones  
 ...

Esto puede hacerse en una misma línea(root):  

    # modprobe nbd max_part=N  

  - 'N' representa el número de particiones que tiene la imagen que vamos a montar  
  Por lo que teniendo esto en cuenta, debe ajustarse con criterio!!  
  Si se trata de una imagen, sin una partición especifica, puede omitirse el  
  parametro.  
 
> Dato sin verificar!

Si el módulo está cargado, lo mejor es descargarlo y cargarlo de nuevo, iniciando la  
variable. En Debian esto parece que tiene un bug. Cuando comprobamos la información  
del módulo(antes y despues de la asignación):  

    # modinfo nbd  

Si está cargado, lo descargamos: 
 
    # rmmod nbd  

...vemos que aparece la ĺinea, pero no el entero! parece un bug. Es la segunda línea  
empezando por abajo. 
> _nota:_ deberías comprobar si en el mailing de Debian se ha escrito el 'report'.  

__antes:__  

filename:       /lib/modules/algo_aqui/kernel/drivers/block/nbd.ko  
license:        GPL  
description:    Network Block Device  
depends:  
intree:         Y  
vermagic:       algo_aqui-tete SMP mod_unload modversions 086  
parm:           nbds_max:number of network block devices to initialize (default: 16) (int)  
parm:           max_part:number of partitions per device (default: 0) (int)  
parm:           debugflags:flags for controlling debug output (int)  

__después:__

    # modprobe nbd max_part=8  

    # modinfo nbd
    filename:    /lib/modules/algo-aqui/kernel/drivers/block/nbd.ko  
    license:     GPL  
    description: Network Block Device  
    depends:  
    intree:      Y  
    vermagic:    algo-aqui SMP mod_unload modversions 086  
    parm:        nbds_max:number of network block devices to initialize(default:16)(int)  
    parm:        max_part:number of partitions per device (default: 0) (int)  
    parm:        debugflags:flags for controlling debug output (int)  


Este comando identifica la imagen, como un dispositivo de bloque llamado  
/dev/nbd0, y la partición dentro de éste, como sub-dispositivo, que sería:  
/dev/nbd0p1.  

    qemu-nbd -c /dev/nbd0 _vdi-file_  

1. CARGAMOS EL MÓDULO  

    # modprobe nbd -- Esto carga el módulo de no estar cargado.  
    # modprobe nbd max_part=16  

2. A continuación preparamos el dispositivo donde montaremos la unidad.  
   Este proceso inicia una especie de servidor. Realmente la carga en memoria es  
   mínima, es decir, no es como si lanzásemos Apache!!!  

    # qemu-nbd -c /dev/nbd0/ /path/to/vhd_file -- Esto conecta el dispositivo.  
    # partprobe /dev/nbd0  -- indica al SO los cambios que se han llevado  
                           a cabo en la tabla de particiones.  

3. Este último paso, es el que realmente monta la unidad virtual en el sistema.  

    # mount /dev/nbd0p1 /imagen/a/montar(vhd en este caso!!)  

> recuerda desmontar la unidad y el dispositivo cuando termines!  

    $ umount /imagen/montada(vhd) -- Desmontamos imagen.  
    # qemu-nbd -d /dev/nbd0 -- desconectamos dispositivo.  


## LANZAR LA VM APUNTANDO AL SERVIDOR NBD  

    $ QEMU -object tls-creds-x509,id=tls0,dir=$HOME/.pki/qemutls,endpoint=client \  
        -drive driver=nbd,host=localhost,port=10809,tls-creds=tls0 \  
        /path/to/img  







http://bethesignal.org/blog/2011/01/05/how-to-mount-virtualbox-vdi-image/ ####

 

Ahora podríamos ejecutar cfdisk en el dispositivo de bloque, y montarlo
como partición individual.  

    # mount /dev/nbd0p1 /mnt -- "/mnt" es el punto de montaje.  



Al terminar podemos desmontar la unidad y el dispositivo, así.  

    $ unmount /mnt  
    # qemu-nbd -d /dev/nbd0  

You can use qemu-nbd in Linux to access a disk image as if it were a block device.  
Here are some examples of operations that can be performed from a live Knoppix terminal.  

    $ su  
    # modprobe nbd  
    # qemu-nbd --read-only --connect=/dev/nbd0 --format=vpc _vhd-file-name_  

If VHDX format:  

    # qemu-nbd --connect=/dev/nbd0 --format=VHDX _hdx-file-name_  
    # ddrescue --verbose --force /dev/nbd0 /dev/sda  # write image to /dev/sda  

Write one partition:  

    # nbd --partition=2 --read-only --connect=/dev/nbd2 --format=vpc vhd-file-name  
    # ddrescue --verbose --force /dev/nbd2 /dev/sda2 # write partition 2 of image to /dev/sda2  


Mount partition:  

    # qemu-nbd --partition=2 --read-only --connect=/dev/nbd2 --format=vpc vhd-file-name  
    # mount /dev/nbd2 /mnt  

Unmount and disconnect image file:  

    $ umount /mnt  
    # qemu-nbd --disconnect /dev/nbd2  

To convert a vhd image to raw (less usable)  

    $ qemu-img convert -f raw -O vpc something.img something.vhd  

To convert a vhd image to cow2 (the up to date qemu format)  

    $ qemu-img convert -f qcow2 -O vpc something.img something.vhd  

---  
##                  E X P E R I M E N T A L  

---  

## <a name="6i">ATAJOS DEL TECLADO</a>
Son combinaciones de teclas, para acceder a funcionalidades própias de Qemu. Generalmaente:  
__ctrl + alt + tecla__:  

  - ctrl+alt: acopla el ratón a la ventanta donde corremos qemu. Pero también las  
    funciones de acceso rápido(teclas vinculadas). Éstas, toman los valores de la máquina  
    en carrera. ejem:  
      VM linux: ctrl+alt+suprimir, cierra sessión.  
      VM windows: ctrl+alt+surpimir, lanza el administrador de procesos.  
    Es decir, hereda las funciones relacionadas con dispositivos de entrada, del proceso  
    emulado/virtualizado.  
  - ctrl+alt+f: maximiza/desmaximiza ventana de la VM.
  - ctrl+alt+u: Recupera el tamaño de la ventana a su dimensión original.
  - ctrl+alt+1: volver al modo en el que hayamos lanzado la VM(gráfico/texto).
  - ctrl+alt+2: Monitor de qemu.
  - ctrl+alt+3: Cónsola en serie.
  - ctrl+alt+4: Cónsola en paralelo. 
  - ctrl+alt+avance página: control de panatalla en qemu monitor y cónsolas.
  - ctrl+alt+retroceso página: control de panatalla en qemu monitor y cónsolas.
  - ctrl+alt+arriba: control de panatalla en qemu monitor y cónsolas.
  - ctrl+alt+abajo: control de panatalla en qemu monitor y cónsolas.

####<a name="6i1"> Comandos del monitor __Qemu__</a>
  - commit device|all: Aplica cambios en imagenes de disco (capturas).
  - info subcommand: información de la VM.  
  - q| quit: cierra qemu.  
  - eject[-f] dispositivo: expulsa un soportes en la supuesta, cd/flopy(probar HD?)
  - change device filename: cambia uno de los soportes de la VM. CD y floppy (HD?)
  - screendump filename: captura de pantalla.
  - savevm ta|id: guarda recupera y borra instantáneas de la supuesta.
  - loadvm tag|id: guarda recupera y borra instantáneas de la supuesta.
  - delvm tag|id: guarda recupera y borra instantáneas de la supuesta.
  - stop / c| cont: detiene y reanuda la emulación.  
  - sendkey keys: envía secuencia de teclas a la VM. Ejem. inicio sesion Windows.
  - `system_reset`: equivale a reset(reboot).
  - `system_powerdown`: equivale a apagado(shutdown).


#### Redefinir teclas:  



## <a name="ai">AGRADECIMIENTOS</a>  

Documentation/Networking --[QEMU][QEM]  
Virtualization Api --[Libvirt][lib]
Manual Capturas(inglés) --[snapshots-handout][fedora]  
ArchWiki --[QEMU][archi]  
Departamento de informática de IPC -[IPC][elpuig]  
Suse --[QEMU][suse]  
Debian --[VLAN][debian]  
IEEE 802.1Q --[Wikia][dot1Q]  
Javier Cristóbal -- [Markdowns y otras recomendaciones][Markdown] \- [productividad mac][limni]  
HeavyMetalRadio [hmr][HMR]

---
[QEM]: http://wiki.qemu.org/Documentation/Networking
[lib]: http://libvirt.org/index.html
[fedora]: https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html
[archi]: https://wiki.archlinux.org/index.php/QEMU#qxl
[elpuig]: http://elpuig.xeill.net/Members/vcarceler/articulos/qemu
[suse]: https://www.suse.com/documentation/sles11/book_kvm/data/cha_qemu_guest_inst_qemu-img.html 
[debian]: https://wiki.debian.org/es/NetworkConfiguration#C.2BAPM-mo_utilizar_VLAN_.28dot1q.2C_802.1q.2C_trunk.29_.28Etch.2C_Lenny.29
[dot1Q]: https://es.wikipedia.org/wiki/IEEE_802.1Q
[Markdown]: http://markdown.es/sintaxis-markdown/
[limni]: http://limni.net/blog/
[HMR]:http://stream.kazancity.net:8000/14-heavymetalradio
