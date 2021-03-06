---
layout: default
title: "Aprende Puppet – Manifiestos"
canonical: "/es/learning/manifests.html"
---

## Comienzo
Has hecho los ejercicios de **puppet resource** del [capítulo anterior]()? Borremos la cuenta que creaste.

En un editor de texto (**vim, emac** o **nano**) crea un archivo con este contenido y nombre:

        # /root/examples/user-absent.pp
        user {'katie':
            ensure => absent,
        }

Sálvalo y cierra el editor. Luego ejecuta:

        # puppet apply /root/examples/user-absent.pp
        notice: /Stage[main]//User[katie]/ensure: removed
        notice: Finished catalog run in 0.07 seconds

Ejecútalo de nuevo:

        # puppet apply /root/examples/user-absent.pp
        notice: Finished catalog run in 0.03 seconds


Genial! Has escrito y aplicado tu primer manifiesto de Puppet.

## Manifiestos
Los programas de Puppet son llamados “manifiestos”, y utilizan la extensión **.pp**. El núcleo del lenguaje de Puppet es la *declaración de recurso*. Una declaración de recurso describe un *estado deseado* para un recurso.

Los manifiestos también pueden utilizar varios tipos de lógica: condicional,  colecciones de recursos, funciones para generar texto, etc. Hablaremos de esto luego.

## Puppet Apply
Como **resource** en el capítulo anterior, **apply** es un subcomando. Toma el nombre de un archivo de manifiesto como su argumento, y aplica el ado descrmanifiesto.

Lo utilizaremos a continuación para probar pequeños manifiestos, pero también puede ser utilizado para trabajos más grandes. De hecho, puede hacer casi todo lo que un entorno agente/master de Puppet puede hacer.

## Declaraciones de recurso
Comencemos observando un solo recurso:

            # /root/examples/file-1.pp

            file {'testfile':
              path    => '/tmp/testfile',
              ensure  => present,
              mode    => 0640,
              content => "I'm a test file.",
            }

[La sintaxis y comportamiento de las declaraciones de recurso están documentados en el manual de referencia de Puppet](), pero en resumen, consiste en:

1. El **Tipo** (**file**, en este caso)
2. Una llave de apertura (**{**)
3. El **título** (**testfile**)
4. Dos puntos (**:**)
5. Un conjunto de pares **atributo=>valor**, con una coma luego de cada par (**path => '/tmp/testfile'**,etc.)
6. Una llave de cierre (})

Intenta aplicar el pequeño manifiesto anterior:

        # puppet apply /root/examples/file-1.pp
        notice: /Stage[main]//File[testfile]/ensure: created
        notice: Finished catalog run in 0.05 seconds

Esto es lo contrario a lo que hemos visto antes cuando borramos la cuenta de usuario: Puppet sabía que ese archivo no existía y lo creó. Estableció el contenido y modo deseado al mismo tiempo.

        # cat /tmp/testfile
        I'm a test file.
        # ls -lah /tmp/testfile
        -rw-r----- 1 root root 16 Feb 23 13:15 /tmp/testfile

Si intentamos cambiar el modo y aplicar el manifiesto nuevamente, Puppet lo reparará.

        # chmod 0666 /tmp/testfile
        # puppet apply /root/examples/file-1.pp
        notice: /Stage[main]//File[testfile]/mode: mode changed '0666' to '0640'
        notice: Finished catalog run in 0.04 seconds

Y si ejecutas el manifiesto nuevamente, verás que Puppet no hace nada; si un recurso ya se encuentra en estado deseado, Puppet lo dejará como está.

**Ejercicio**: Declara otro recurso de archivo en un manifiesto y aplícalo. Intenta establecer un nuevo estado deseado para un archivo existente, por ejemplo, cambia el mensaje de login definiendo el contenido de **/etc/motd**. Puedes [ver los atributos disponibles para el tipo de archivo aquí]().

### Hints de Sintaxis
Observa estos errores frecuentes:

+ No olvides las comas y los dos puntos! Si los olvidas causarás errores del tipo **Could not parse for environment production: Syntax error at 'mode'; expected '}' at /root/manifests/1.file.pp:6 on node learn.localdomain**.
+ El uso de mayúsculas y minúsculas es importante! El tipo de recurso y el nombre de atributo deben estar siempre en minúscula.
+ Los valores utilizados para títulos y valores de atributos normalmente serán [strings](), los cuales normalmente debes colocar entre comillas. [Lee más acerca de los tipos de datos de Puppet aquí]().
        + Existen dos tipos de comillas en Puppet: simple (**’**) y doble (**”**). La mayor diferencia es que la doble comilla te permite interpolar $variables, de las que hablaremos en otra clase.
        + Los nombres de atributos (como **path**, **ensure**, etc.) son claves especiales, no strings. Las mismas no necesitan comillas.

También ten en cuenta que Puppet te permite usar espacios en blanco para hacer más legible tus manifiestos. Nosotros sugerimos alinear las flechas **=>** ya que esto facilita entender un manifiesto a simple vista. Los plugins de Vim en la VM de Aprende Puppet harán esto automáticamente mientras tipees.

## Una vez más!
Ahora que conoces las declaraciones de recurso, juguemos un poco más con el tipo de archivo:

+ Pondremos múltiples recursos de tipos diferentes en el mismo manifiesto
+ Utilizaremos valores nuevos para el atributo **ensure**
+ Encontraremos un atributo con una relación especial con el título del recurso
+ Veremos qué sucede cuando no especificamos algunos atributos.
+ Veremos algunos ajustes de permisos automáticos en directorios

<!-- -->

    # /root/examples/file-2.pp

    file {'/tmp/test1':
      ensure  => file,
      content => "Hi.",
    }

    file {'/tmp/test2':
      ensure => directory,
      mode   => 0644,
    }

    file {'/tmp/test3':
      ensure => link,
      target => '/tmp/test1',
    }

    user {'katie':
      ensure => absent,
    }

    notify {"I'm notifying you.":}
    notify {"So am I!":}

Aplica:

    # puppet apply /root/examples/file-2.pp
    notice: /Stage[main]//File[/tmp/test1]/ensure: created
    notice: /Stage[main]//File[/tmp/test3]/ensure: created
    notice: /Stage[main]//File[/tmp/test2]/ensure: created
    notice: So am I!
    notice: /Stage[main]//Notify[So am I!]/message: defined 'message' as 'So am I!'
    notice: I'm notifying you.
    notice: /Stage[main]//Notify[I'm notifying you.]/message: defined 'message' as 'I'm notifying you.'
    notice: Finished catalog run in 0.05 seconds

Genial. Qué ha pasado?

### Nuevos valores para ensure, estados diferentes
El atributo **ensure** es algo especial. Está disponible en casi todos los tipos de recursos y controla si existe el recurso, siendo la definición “existe” algo local.

Con archivos, hay varias maneras de existir:

1. Como un archivo normal (**ensure => file**)
2. Como un directorio (**ensure => directory**)
3. Como un link simbólico (**ensure => link**)
4. Como cualquiera de los anteriores (**ensure => present**)
5. Como *nada* (**ensure => absent**).

Un chequeo rápido muestra cómo se desarrolla el manifiesto

        # ls -lah /tmp/test
        -rw-r--r--  1 root root    3 Feb 23 15:54 test1
        lrwxrwxrwx  1 root root   10 Feb 23 15:54 test3 -> /tmp/test1

        /tmp/test2:
        total 16K
        drwxr-xr-x 2 root root 4.0K Feb 23 16:02 .
        drwxrwxrwt 5 root root 4.0K Feb 23 16:02 ..

        # cat /tmp/test3
        Hi.

### Títulos y namevars
¿Has notado que el recurso de archivo original tenía un atributo **path** pero, los siguientes tres lo omitieron?

Casi todo tipo de recurso tiene un atributo cuyo valor se asume por defecto como el título del recurso. Para el recurso **file**, ese atributo es **path**. La mayoría de las veces (**user**, **group**, **package**…) es **name**.

Estos atributos se llaman **namevars**; son generalmente el atributo que corresponde a la *identidad* del recurso, la única cosa que siempre debería ser única. Si omites el namevar de un recurso, Puppet reutilizará el título como su valor. En caso que especifiques un valor para un namevar, el título del recurso puede ser cualquier cosa.

#### Identidad e identidad
Entonces, para qué tener un namevar si Puppet puede reutilizar un título?

Hay dos tipos de identidad que Puppet reconoce:

1. Identidad dentro de Puppet
2. Identidad en el sistema de destino

La mayoría de las veces son la misma, pero a veces no. Por ejemplo, el servicio NTP tiene diferente nombre en plataformas diferentes: En sistemas como Red Hat, se llama **ntpd**; y en sistemas como Debian, **ntp**. Lógicamente son el mismo recurso, pero su identidad en el sistema de destino no es el mismo.

También hay casos (generalmente recursos **exec**) donde la identidad del sistema no tiene un significado particular, y colocar una identidad más descriptiva en el título puede ayudar para contarle a tus colegas (o a ti mismo en dos meses) qué debería hacer ese recurso.

Al permitirte separar el título y el namevar, Puppet facilita el manejo de esos casos. Hablaremos de esto luego cuando lleguemos a las declaraciones condicionales.

#### Unicidad
Ten en cuenta que no puedes declarar el mismo recurso dos veces. Puppet nunca permite duplicar *títulos* dentro de un tipo determinado, y generalmente tampoco permite duplicar *valores de namevar* en un tipo.

Esto es porque las declaraciones de recursos representan estados deseados finales, y no queda para nada claro lo que debe suceder si declaras dos estados en conflicto. Entonces Puppet generará un error en lugar de hacer algo mal por accidente en el sistema.

### Atributos faltantes: “Estado deseado=Lo que sea”
En el archivo **/tmp/test1** omitimos los atributos **mode** y **owner**, entre otros. Cuando omitimos atributos, Puppet no los maneja, y se asume un valor cualquiera como estado deseado.

Si un archivo no existe, Puppet lo creará por defecto con modo de permisos 0644, pero si cambias ese modo, Puppet no lo volverá a cambiar.

Ten en cuenta que incluso puedes omitir el atributo **ensure** siempre que no especifiques **content** o **source**. Esto te permite manejar los permisos de un archivo existente, no pero te permite crearlo si no existe.

### Permisos de directorio: 644 = 755
Dijimos que **/tmp/test2/** debe tener un modo de permiso 0644, pero **ls –lah** muestra el modo 0755. Esto se debe a que Puppet agrupa el bit de lectura y el bit de traverso de directorios.

Esto ayuda al manejo de directorios (con **recurse => true**), con lo que puedes permitir el recorrido de los directorios sin hacer ejecutable todo el contenido del directorio.

## Destinos, no viajes
Ya sabes que hablamos mucho acerca de “estados deseados”, en lugar de hablar de cambios en el sistema. Esto es el fundamento del pensamiento del usuario de Puppet.

Si estuvieras escribiendo una explicación para otra persona sobre cómo llevar un sistema a un estado deseado utilizando las herramientas que OS tiene por defecto, escribirías algo como “Chequea que el modo del archivo sudoers sea 0440, usando **ls –l** Si con eso es suficiente, ve al siguiente paso; de lo contrario, ejecuta **chmod 0440 /etc/sudoers**.

Internamente, Puppet está haciendo lo mismo, con algunas de las mismas herramientas de OS, pero une los pasos “chequeo” y “y arregla si es necesario” ,y los presenta como una sola interface.

El efecto es ese: en lugar de escribir un script bash que parece un paso a paso para un principiante, puedes escribir manifiestos de Puppet que se vean como notas para un experto.

## Nota aparte: Compilación
Los manifiestos no son usados directamente cuando Puppet sincroniza recursos. Por el contrario, el flujo de una ejecución de Puppet es más o menos así:

![](img/manifest_to_defined_state_unified.png)

Como hemos mencionado antes, los manifiestos pueden contener declaraciones condicionales, variables, funciones y otras formas de lógica; pero antes de aplicarse, los manifiestos se *compilan* en un documento llamado **”catálogo”** el cual *sólo* contiene recursos y pistas acerca del orden para sincronizarlos.

Con Puppet Apply, la distinción no es importante, pero en un entorno master/agente de Puppet sí lo es porque los agentes sólo ven el catálogo:

+ Utilizando lógica, los manifiestos pueden ser flexibles y describir muchos sistemas de una sola vez. Un catálogo describe estados deseados para **un** sistema.
+ Por defecto, los nodos agentes sólo pueden recuperar su propio catálogo; no pueden ver información destinada a otros nodos. Esta separación mejora la seguridad.
+ Como los catálogos son tan poco ambiguos, es posible *simular* la ejecución de un catálogo sin hacer ningún cambio en el sistema; esto se realiza generalmente ejecutando **puppet agent --test –noop** Puedes incluso utilizar herramientas de diff especiales para comparar dos catálogos y ver las diferencias.

## El manifiesto del sitio y el agente de Puppet
Hemos visto cómo utilizar Puppet Apply para aplicar en forma directa manifiestos en un sistema. Los servicios de master/agente de Puppet funcionan de manera muy similar, pero con algunas diferencias clave:

**Puppet apply**:

+ Un usuario ejecuta un comando, activando una ejecución de Puppet
+ Puppet Apply lee los manifiestos pasados a él, los compila en un catálogo y lo aplica.

**Agente/master**:

+ El agente de Puppet ejecuta un servicio y activa la ejecución de Puppet cada media hora (esto es configurable)
        + En tu VM, la cual utiliza Puppet Enterprise, el servicio agente se llama **pe –puppet**. (Puppet agent puede también configurarse para ejecutarse desde cron en lugar de un servicio)
+ El agente no tiene acceso a ningún manifiesto. Por el contrario, requiere de un catálogo previamente compilado en un servidor Puppet Master.
        + En tu VM, el Puppet Master aparece como el servicio **pe-httpd**, una copia aislada de Apache con Passenger maneja la aplicación de Puppet Master generando y eliminando tantas copias como necesite.
+ El Puppet Master siempre lee **un** manifiesto especial, llamado el “manifiesto del sitio” o site.pp. Utiliza esto para compilar un catálogo, el cual se envía al agente.
        + En tu VM, el manifiesto del sitio está en **/etc/puppetlabs/puppet/manifests/site.pp**
+ Luego de obtener el catálogo, el agente lo aplica.

De esta forma, puedes tener muchas máquinas configuradas por Puppet con sólo mantener tus manifiestos en uno o varios servidores. Esto también te da seguridad extra, como fue descrito antes en “Compilación”.

### Ejercicio: Utiliza Puppet Master/Agente para aplicar la misma configuración
Para ver cómo funciona el mismo código de manifiesto con el agente de Puppet:

+ Edita **/etc/puppetlabs/puppet/manifests/site.pp** y pega los tres recursos de archivo del manifiesto anterior.
        + Ten cuidado con parte del código existente en site.pp, y todavía no toques ninguna declaración de **nodo**. Puedes pegar los recursos al final del archivo y funcionarán bien.
+ Borra o corta los archivos y directorios creados en **/tmp**
+ Ejecuta **puppet agent --test**, el cual activará solo una ejecución agente puppet en primer plano para que puedas ver que está haciendo en tiempo real.
+ Chequea **/tmp** y fíjate que los archivos volvieron a su estado deseado.

### Ejercicio: Clave autorizada SSH
Escribe y aplica un manifiesto que utilice el [tipo ssh\_authorized\_key]() para que te permita loguearte en la VM de práctica como *root* sin contraseña.

**Trabajo extra**: Intenta ponerlo directamente en el manifiesto del sitio, en lugar de utilizar Puppet Apply, [utiliza la consola para activar la ejecución del agente de Puppet]() y [chequea los reportes en la consola]() para ver si ha funcionado el manifiesto.

+ Necesitarás tener un par de claves de SSH, una aplicación de terminal en tu servidor, y conocimientos básicos acerca del funcionamiento de SSH. Puedes obtener esto investigando un poco.
+ Cuidado: No puedes pegar la línea de **id_rsa.pub** en el atributo **clave** del recurso. Necesitarás separar sus componentes en atributos múltiples. Lee la documentación del tipo **ssh\_authorized\_key** para enterarte cómo.

## Siguiente
**Próxima clase:**
Sabes cómo utilizar los bloques de construcción fundamentales de código Puppet. Así que ahora es momento de aprender cómo combinarlos.

**Nota aparte:**
Ya sabes básicamente cómo trabajar con Puppet; manejar la pertenencia de archivos y permisos es importante.

Hay muchos archivos en tus sistemas que has intentado sincronizar? Ya sabes suficiente como para ocuparte de ellos. [Descarga Puppet Enterprise](), sigue [la guía rápida de comienzo]() para instalar un pequeño entorno, y luego intenta poner algunos recursos de archivo al final del archivo **site.pp** de Puppet Master para manejar esos archivos en cada máquina.
