# Git y Otros Sistemas #

El mundo no es perfecto. Lo más normal es que no puedas cambiar inmediatamente a Git cada proyecto que te encuentras. Algunas veces estás atascado en un proyecto utilizando otro VCS, y muchas veces ese sistema es Subversion. Pasaremos la primera parte de este capítulo aprendiendo sobre `git svn`, la puerta de enlace de Subversion bidireccional en Git.

En un momento dado, quizá quieras convertir tu proyecto existente a Git. La segunda parte de este capítulo cubre cómo migrar tu proyecto a Git: primero desde Subversion, luego desde Perforce, y finalmente por medio de un script de importación a medida para casos no estándar.

## Git y Subversion ##

Hoy por hoy, la mayoría de los proyectos de código abierto y un gran número de proyectos corporativos usan Subversion para manejar su código fuente. Es el VCS más popular y lleva ahí casi una década. También es muy similar en muchas cosas a CVS, que fue el rey del control de código fuente anteriormente.

Una de las grandes características de Git es el puente bidireccional llamado `git svn`. Esta herramienta permite el uso de Git como un cliente válido para un servidor Subversion, así que puedes utilizar todas las características locales de Git y luego hacer publicaciones al servidor de Subversion como si estuvieras usando Subversion localmente. Esto significa que puedes ramificar y fusionar localmente, usar el área de preparación, reconstruir, entresacar, etc., mientras tus colaboradores continúan usando sus antiguos y oscuros métodos. Es una buena forma de colar a Git dentro de un ambiente corporativo y ayudar a tus colegas desarrolladores a hacerse más eficientes mientras tu haces presión para que se cambie la infraestructura y el soporte de Git sea completo. El puente de Subversion es la "droga de entrada" al mundo de DVCS.

### git svn ###

El comando básico de Git para todos los comandos de enlace con Subversion es `git svn`. Siempre debes empezar con eso. Hay unos cuantos, por lo que vamos a aprender los básicos recorriendo unos pocos flujos de trabajo pequeños.

Es importante fijarse en que cuando usas `git svn` estás interactuando con Subversion, que es un sistema mucho menos sofisticado que Git. Aunque puedes hacer ramas y fusiones localmente, lo mejor es mantener tu historia lo más lineal posible mediante reorganizaciones, y evitar hacer cosas como interactuar a la vez con un repositorio remoto de Git.

No reescribas tu historia y trates de publicar de nuevo, y no publiques a un repositorio Git paralelo para colaborar con colegas que usen Git al mismo tiempo. Subversion sólo puede tener una historia lineal simple, de lo contrario puedes confundirlo fácilmente. Si trabajas con un equipo, y algunos usan SVN mientras otros usan Git, asegúrate de que todos usan el servidor de SVN para colaborar. Si lo haces así, tu vida será más simple.

### Configuración ###
Para demostrar esta funcionalidad, se necesita un repositorio SVN típico al que tengamos acceso de escritura. Si se desea copiar estos ejemplos, tendrás que crear una copia grabable de mi repositorio de pruebas. Con el fin de hacerlo con facilidad, puede utilizar una herramienta llamada `svnsync` que viene con versiones más recientes de Subversion - que debe ser distribuido con al menos la versión 1,4. Para estas pruebas, creé un nuevo repositorio de Subversion en el código de Google que era una copia parcial del proyecto  `protobuf` , que es una herramienta que codifica datos estructurados para la transmisión de red.

Para seguir adelante, primero se debe crear un nuevo repositorio local de Subversion:

	$ mkdir /tmp/test-svn
	$ svnadmin create /tmp/test-svn

A continuación, habilite a todos los usuarios a cambiar revprops - la forma más sencilla es agregar un script de pre-revprop-change que siempre sale de 0:

	$ cat /tmp/test-svn/hooks/pre-revprop-change 
	#!/bin/sh
	exit 0;
	$ chmod +x /tmp/test-svn/hooks/pre-revprop-change

Ahora puede sincronizar este proyecto con su máquina local llamando `svnsync init` con los repositorios de y hacia.

	$ svnsync init file:///tmp/test-svn http://progit-example.googlecode.com/svn/ 

Esto configura las propiedades para ejecutar la sincronización. A continuación, puede clonar el código ejecutando

	$ svnsync sync file:///tmp/test-svn
	Committed revision 1.
	Copied properties for revision 1.
	Committed revision 2.
	Copied properties for revision 2.
	Committed revision 3.
	...

Aunque esta operación puede tardar sólo unos minutos, si intenta copiar el repositorio original en otro repositorio remoto en lugar de uno local, el proceso tardará casi una hora, aunque hay menos de 100 confirmaciones. Subversion tiene que clonar una revisión a la vez y luego empujarla de nuevo a otro repositorio - es ridículamente ineficiente, pero es la única manera fácil de hacer esto.

### Empezando ###

Ahora que tiene un repositorio de Subversion al que tiene acceso de escritura, puede pasar por un flujo de trabajo típico. Comenzarás con el comando `git svn clone`, que importa todo un repositorio de Subversion en un repositorio Git local. Recuerde que si está importando desde un repositorio real de Subversion alojado, debe reemplazar el `file: /// tmp / test-svn` aquí con la URL de su repositorio de Subversion:

	$ git svn clone file:///tmp/test-svn -T trunk -b branches -t tags
	Initialized empty Git repository in /Users/schacon/projects/testsvnsync/svn/.git/
	r1 = b4e387bc68740b5af56c2a5faf4003ae42bd135c (trunk)
	      A    m4/acx_pthread.m4
	      A    m4/stl_hash.m4
	...
	r75 = d1957f3b307922124eec6314e15bcda59e3d9610 (trunk)
	Found possible branch point: file:///tmp/test-svn/trunk => \
	    file:///tmp/test-svn /branches/my-calc-branch, 75
	Found branch parent: (my-calc-branch) d1957f3b307922124eec6314e15bcda59e3d9610
	Following parent with do_switch
	Successfully followed parent
	r76 = 8624824ecc0badd73f40ea2f01fce51894189b01 (my-calc-branch)
	Checked out HEAD:
	 file:///tmp/test-svn/branches/my-calc-branch r76

Esto ejecuta el equivalente a dos comandos - `git svn init` seguido de` git svn fetch` - en la URL que usted proporciona. Esto puede tomar un tiempo. El proyecto de prueba tiene sólo unos 75 compromisos y la base de código no es tan grande, por lo que sólo toma unos minutos. Sin embargo, Git tiene que revisar cada versión, una a la vez, y confiar individualmente. Para un proyecto con cientos o miles de compromisos, esto puede tomar literalmente horas o incluso días para terminar.

La parte `-T trunk -b branches -t tags` le dice a Git que este repositorio de Subversion sigue las convenciones básicas de ramificación y etiquetado. Si nombra el tronco, las ramas o las etiquetas de forma diferente, puede cambiar estas opciones. Debido a que esto es tan común, puede reemplazar esta parte entera con `-s`, lo que significa un diseño estándar e implica todas esas opciones. El siguiente comando es equivalente:

	$ git svn clone file:///tmp/test-svn -s

En este punto, debe tener un repositorio Git válido que haya importado sus ramas y etiquetas:

	$ git branch -a
	* master
	  my-calc-branch
	  tags/2.0.2
	  tags/release-2.0.1
	  tags/release-2.0.2
	  tags/release-2.0.2rc1
	  trunk

Es importante tener en cuenta cómo esta herramienta de nombres de sus referencias remotas de manera diferente. Cuando está clonando un repositorio Git normal, obtiene todas las ramas de ese servidor remoto disponibles localmente como algo similar a `origen / [rama]` - con el nombre del control remoto. Sin embargo, `git svn` supone que no tendrá múltiples controles remotos y guarda todas sus referencias a puntos en el servidor remoto sin ningún espacio de nombres. Puede utilizar el comando git plumbing `show-ref` para ver todos los nombres de referencia completos:

	$ git show-ref
	1cbd4904d9982f386d87f88fce1c24ad7c0f0471 refs/heads/master
	aee1ecc26318164f355a883f5d99cff0c852d3c4 refs/remotes/my-calc-branch
	03d09b0e2aad427e34a6d50ff147128e76c0e0f5 refs/remotes/tags/2.0.2
	50d02cc0adc9da4319eeba0900430ba219b9c376 refs/remotes/tags/release-2.0.1
	4caaa711a50c77879a91b8b90380060f672745cb refs/remotes/tags/release-2.0.2
	1c4cb508144c513ff1214c3488abe66dcb92916f refs/remotes/tags/release-2.0.2rc1
	1cbd4904d9982f386d87f88fce1c24ad7c0f0471 refs/remotes/trunk

Un repositorio normal de Git se parece más a esto:

	$ git show-ref
	83e38c7a0af325a9722f2fdc56b10188806d83a1 refs/heads/master
	3e15e38c198baac84223acfc6224bb8b99ff2281 refs/remotes/gitserver/master
	0a30dd3b0c795b80212ae723640d4e5d48cabdff refs/remotes/origin/master
	25812380387fdd55f916652be4881c6f11600d6f refs/remotes/origin/testing

Tiene dos servidores remotos: uno llamado `gitserver` con una rama` master`; Y otro llamado `origen` con dos ramas,` master` y `testing`.

Observe cómo en el ejemplo de referencias remotas importadas de `git svn`, las etiquetas se agregan como ramas remotas, no como etiquetas Git reales. Su importación de Subversion parece que tiene un remoto llamado etiquetas con ramas bajo él.

### Volviendo a Comitear en Subversion ###

Ahora que tiene un repositorio de trabajo, puede hacer algún trabajo en el proyecto y empujar sus confirmaciones de nuevo en sentido ascendente, utilizando Git eficazmente como un cliente SVN. Si edita uno de los archivos y lo confirma, tiene un commit que existe en Git localmente que no existe en el servidor de Subversion:

	$ git commit -am 'Adding git-svn instructions to the README'
	[master 97031e5] Adding git-svn instructions to the README
	 1 files changed, 1 insertions(+), 1 deletions(-)

A continuación, debe empujar su cambio hacia arriba. Observe cómo esto cambia la forma en que trabaja con Subversion: puede realizar varios commit offline y luego empujarlos todos de una vez al servidor de Subversion. Para enviar un servidor Subversion, ejecute el comando `git svn dcommit`:

	$ git svn dcommit
	Committing to file:///tmp/test-svn/trunk ...
	       M      README.txt
	Committed r79
	       M      README.txt
	r79 = 938b1a547c2cc92033b74d32030e86468294a5c8 (trunk)
	No changes between current HEAD and refs/remotes/trunk
	Resetting to the latest refs/remotes/trunk

Esto toma todos los commits que has hecho en la parte superior del código del servidor de Subversion, hace un commit de Subversion para cada uno, y luego vuelve a escribir tu commit Git local para incluir un identificador único. Esto es importante porque significa que todas las sumas de comprobación SHA-1 para su confirmación cambian. En parte por esta razón, trabajar con versiones remotas basadas en Git de sus proyectos simultáneamente con un servidor Subversion no es una buena idea. Si observa el último commit, puede ver el nuevo `git-svn-id` que se agregó:

	$ git log -1
	commit 938b1a547c2cc92033b74d32030e86468294a5c8
	Author: schacon <schacon@4c93b258-373f-11de-be05-5f7a86268029>
	Date:   Sat May 2 22:06:44 2009 +0000

	    Adding git-svn instructions to the README

	    git-svn-id: file:///tmp/test-svn/trunk@79 4c93b258-373f-11de-be05-5f7a86268029

Observe que la suma de comprobación SHA que comenzó originalmente con `97031e5` cuando se comprometió ahora comienza con` 938b1a5`. Si desea enviar tanto a un servidor Git como a un servidor Subversion, primero tiene que empujar (`dcommit`) al servidor de Subversion, porque esa acción cambia los datos de confirmación.

### Sacando nuevos cambios ###

Si está trabajando con otros desarrolladores, en algún momento uno de ustedes empujará, y luego el otro intentará empujar un cambio que entre en conflicto. Ese cambio será rechazado hasta que se fusionen en su trabajo. En `git svn`, se ve así:

	$ git svn dcommit
	Committing to file:///tmp/test-svn/trunk ...
	Merge conflict during commit: Your file or directory 'README.txt' is probably \
	out-of-date: resource out of date; try updating at /Users/schacon/libexec/git-\
	core/git-svn line 482

Para resolver esta situación, puede ejecutar `git svn rebase`, que anula todos los cambios en el servidor que todavía no tiene y rebases cualquier trabajo que tenga encima de lo que está en el servidor:

	$ git svn rebase
	       M      README.txt
	r80 = ff829ab914e8775c7c025d741beb3d523ee30bc4 (trunk)
	First, rewinding head to replay your work on top of it...
	Applying: first user change

Ahora, todo su trabajo está encima de lo que está en el servidor de Subversion, por lo que puede `dcommit`:

	$ git svn dcommit
	Committing to file:///tmp/test-svn/trunk ...
	       M      README.txt
	Committed r81
	       M      README.txt
	r81 = 456cbe6337abe49154db70106d1836bc1332deed (trunk)
	No changes between current HEAD and refs/remotes/trunk
	Resetting to the latest refs/remotes/trunk

Es importante recordar que, a diferencia de Git, que requiere que se fusione el trabajo de upstream que aún no tienen localmente antes de poder empujar, `git svn` hace que lo haga sólo si los cambios de conflicto. Si alguien empuja un cambio a un archivo y luego presiona un cambio en otro archivo, su `dcommit` funcionará bien:

	$ git svn dcommit
	Committing to file:///tmp/test-svn/trunk ...
	       M      configure.ac
	Committed r84
	       M      autogen.sh
	r83 = 8aa54a74d452f82eee10076ab2584c1fc424853b (trunk)
	       M      configure.ac
	r84 = cdbac939211ccb18aa744e581e46563af5d962d0 (trunk)
	W: d2f23b80f67aaaa1f6f5aaef48fce3263ac71a92 and refs/remotes/trunk differ, \
	  using rebase:
	:100755 100755 efa5a59965fbbb5b2b0a12890f1b351bb5493c18 \
	  015e4c98c482f0fa71e4d5434338014530b37fa6 M   autogen.sh
	First, rewinding head to replay your work on top of it...
	Nothing to do.

Esto es importante recordar, porque el resultado es un estado del proyecto que no existía en ninguno de sus equipos cuando lo empujó. Si los cambios son incompatibles pero no entran en conflicto, es posible que obtenga problemas difíciles de diagnosticar. Esto es diferente a usar un servidor Git. En Git, puede probar completamente el estado en su sistema cliente antes de publicarlo, mientras que en SVN, no puede estar seguro de que los estados inmediatamente antes de commit y después de commit son idénticos.

También debe ejecutar este comando para extraer los cambios del servidor de Subversion, incluso si no está listo para comprometerse. Puede ejecutar `git svn fetch` para capturar los nuevos datos, pero` git svn rebase` hace la búsqueda y luego actualiza sus commits locales.

	$ git svn rebase
	       M      generate_descriptor_proto.sh
	r82 = bd16df9173e424c6f52c337ab6efa7f7643282f1 (trunk)
	First, rewinding head to replay your work on top of it...
	Fast-forwarded master to refs/remotes/trunk.

Ejecutar `git svn rebase` de vez en cuando se asegura de que su código esté siempre actualizado. Sin embargo, debe asegurarse de que su directorio de trabajo esté limpio al ejecutarlo. Si tiene cambios locales, debe almacenar su trabajo o confirmarlo temporalmente antes de ejecutar `git svn rebase`; de lo contrario, el comando se detendrá si ve que el rebase resultará en un conflicto de combinación.

### Incidencias en las ramificaciones de Git ###

Cuando te sientas cómodo con un flujo de trabajo de Git, es probable que crees ramas de temas, trabajes en ellas y luego las fusiones. Si estás empujando a un servidor de Subversion a través de git svn, tal vez quieras rebasar tu trabajo En una sola rama cada vez en lugar de fusionar ramas entre sí. La razón por la que prefiere rebases es que Subversion tiene una historia lineal y no se ocupa de fusiones como Git, por lo que git svn sigue solo al primer padre cuando convierte las instantáneas en commit de Subversion.



	$ git svn dcommit
	Committing to file:///tmp/test-svn/trunk ...
	       M      CHANGES.txt
	Committed r85
	       M      CHANGES.txt
	r85 = 4bfebeec434d156c36f2bcd18f4e3d97dc3269a2 (trunk)
	No changes between current HEAD and refs/remotes/trunk
	Resetting to the latest refs/remotes/trunk
	COPYING.txt: locally modified
	INSTALL.txt: locally modified
	       M      COPYING.txt
	       M      INSTALL.txt
	Committed r86
	       M      INSTALL.txt
	       M      COPYING.txt
	r86 = 2647f6b86ccfcaad4ec58c520e369ec81f7c283c (trunk)
	No changes between current HEAD and refs/remotes/trunk
	Resetting to the latest refs/remotes/trunk

Ejecutar `dcommit` en una sucursal con un historial combinado funciona bien, excepto que cuando miras tu historial de proyectos de Git, no ha vuelto a escribir ninguno de los commit que has hecho en la rama` experimento`. SVN versión de la confirmación de combinación única.

Cuando alguien clones ese trabajo, todo lo que ven es la fusión comen con todo el trabajo aplastado en él; No ven los datos de confirmación de dónde vino o cuándo se cometió.

### Ramificación de Subversion ###

Ramificar en Subversion no es lo mismo que ramificar en Git; Si puedes evitar usarlo mucho, probablemente sea mejor. Sin embargo, puede crear y comprometerse a las sucursales en Subversion mediante git svn.

#### Creando una nueva rama SVN ####

Para crear una nueva sucursal en Subversion, ejecuta `git svn branch [nombre de rama]`:

	$ git svn branch opera
	Copying file:///tmp/test-svn/trunk at r87 to file:///tmp/test-svn/branches/opera...
	Found possible branch point: file:///tmp/test-svn/trunk => \
	  file:///tmp/test-svn/branches/opera, 87
	Found branch parent: (opera) 1f6bfe471083cbca06ac8d4176f7ad4de0d62e5f
	Following parent with do_switch
	Successfully followed parent
	r89 = 9b6fe0b90c5c9adf9165f700897518dbc54a7cbf (opera)

Esto hace el equivalente al comando `svn copy trunk branches / opera` en Subversion y opera en el servidor de Subversion. Es importante tener en cuenta que no te echa un vistazo a esa rama; Si te comprometes en este punto, ese commit irá a `trunk` en el servidor, no` opera`.

### Cambio de ramas activas ###

Git calcula hacia fuera qué rama sus dcommits van a buscar la extremidad de cualesquiera de sus ramificaciones de Subversion en su historia - usted debe tener solamente uno, y debe ser el último con `git-svn-id` en su rama actual historia.

Si desea trabajar en más de una rama simultáneamente, puede configurar ramas locales en `dcommit` a ramas específicas de Subversion iniciándolas en el commit Subversion importado para esa rama. Si desea una rama `opera` en la que pueda trabajar por separado, puede ejecutar

	$ git branch opera remotes/opera

Ahora, si desea fusionar su rama `opera` en` trunk` (su rama `master`), puede hacerlo con una` git merge` normal. Pero necesitas proporcionar un mensaje de confirmación descriptivo (a través de `-m`), o la fusión dirá" Fusionar ópera de rama "en lugar de algo útil.

Recuerde que aunque está usando `git merge` para hacer esta operación, y la fusión probablemente será mucho más fácil de lo que sería en Subversion (porque Git detectará automáticamente la base de combinación apropiada para usted), esto no es normal Git merge commit. Tienes que volver a enviar estos datos a un servidor de Subversion que no puede manejar un commit que rastree más de un padre; Así que, después de empujarlo hacia arriba, se verá como un solo commit que se aplastó en todo el trabajo de otra rama en un solo commit. Después de combinar una rama en otra, no puede volver fácilmente y seguir trabajando en esa rama, como normalmente lo hace en Git. El comando `dcommit` que ejecuta borra cualquier información que indique en qué rama se fusionó, de modo que los cálculos posteriores de la base de combinación se equivocan - el dcommit hace que su resultado` git merge` parezca ejecutado `git merge --squash`. Desafortunadamente, no hay una buena manera de evitar esta situación - Subversion no puede almacenar esta información, por lo que siempre estará lisiado por sus limitaciones mientras lo está utilizando como su servidor. Para evitar problemas, debe eliminar la rama local (en este caso, `opera`) después de fusionarla en el tronco.

### Comandos de Subversion ###

El conjunto de herramientas `git svn` proporciona una serie de comandos para facilitar la transición a Git proporcionando una funcionalidad similar a la que tenía en Subversion. Aquí hay algunos comandos que le dan a lo que Subversion solía hacerlo.

#### Historia del estilo de SVN ####

Si estás acostumbrado a Subversion y quieres ver tu historia en el estilo de salida de SVN, puedes ejecutar `git svn log` para ver tu historial de confirmación en formato SVN:

	$ git svn log
	------------------------------------------------------------------------
	r87 | schacon | 2009-05-02 16:07:37 -0700 (Sat, 02 May 2009) | 2 lines

	autogen change

	------------------------------------------------------------------------
	r86 | schacon | 2009-05-02 16:00:21 -0700 (Sat, 02 May 2009) | 2 lines

	Merge branch 'experiment'

	------------------------------------------------------------------------
	r85 | schacon | 2009-05-02 16:00:09 -0700 (Sat, 02 May 2009) | 2 lines
	
	updated the changelog

Debes saber dos cosas importantes acerca de `git svn log`. En primer lugar, funciona fuera de línea, a diferencia del comando real `svn log`, que solicita al servidor Subversion los datos. En segundo lugar, sólo muestra los compromisos que se han comprometido hasta el servidor de Subversion. Local Git se compromete a que usted no ha dcommited no aparecen; Ninguno de los compromisos que las personas han hecho en el servidor de Subversion en el ínterin. Es más como el último estado conocido de los commits en el servidor de Subversion.

#### SVN Annotation ####

Much as the `git svn log` command simulates the `svn log` command offline, you can get the equivalent of `svn annotate` by running `git svn blame [FILE]`. The output looks like this:

	$ git svn blame README.txt 
	 2   temporal Protocol Buffers - Google's data interchange format
	 2   temporal Copyright 2008 Google Inc.
	 2   temporal http://code.google.com/apis/protocolbuffers/
	 2   temporal 
	22   temporal C++ Installation - Unix
	22   temporal =======================
	 2   temporal 
	79    schacon Committing in git-svn.
	78    schacon 
	 2   temporal To build and install the C++ Protocol Buffer runtime and the Protocol
	 2   temporal Buffer compiler (protoc) execute the following:
	 2   temporal 

Again, it doesn’t show commits that you did locally in Git or that have been pushed to Subversion in the meantime.

#### SVN Server Information ####

You can also get the same sort of information that `svn info` gives you by running `git svn info`:

	$ git svn info
	Path: .
	URL: https://schacon-test.googlecode.com/svn/trunk
	Repository Root: https://schacon-test.googlecode.com/svn
	Repository UUID: 4c93b258-373f-11de-be05-5f7a86268029
	Revision: 87
	Node Kind: directory
	Schedule: normal
	Last Changed Author: schacon
	Last Changed Rev: 87
	Last Changed Date: 2009-05-02 16:07:37 -0700 (Sat, 02 May 2009)

This is like `blame` and `log` in that it runs offline and is up to date only as of the last time you communicated with the Subversion server.

#### Ignoring What Subversion Ignores ####

If you clone a Subversion repository that has `svn:ignore` properties set anywhere, you’ll likely want to set corresponding `.gitignore` files so you don’t accidentally commit files that you shouldn’t. `git svn` has two commands to help with this issue. The first is `git svn create-ignore`, which automatically creates corresponding `.gitignore` files for you so your next commit can include them.

The second command is `git svn show-ignore`, which prints to stdout the lines you need to put in a `.gitignore` file so you can redirect the output into your project exclude file:

	$ git svn show-ignore > .git/info/exclude

That way, you don’t litter the project with `.gitignore` files. This is a good option if you’re the only Git user on a Subversion team, and your teammates don’t want `.gitignore` files in the project.

### Git-Svn Summary ###

The `git svn` tools are useful if you’re stuck with a Subversion server for now or are otherwise in a development environment that necessitates running a Subversion server. You should consider it crippled Git, however, or you’ll hit issues in translation that may confuse you and your collaborators. To stay out of trouble, try to follow these guidelines:

* Keep a linear Git history that doesn’t contain merge commits made by `git merge`. Rebase any work you do outside of your mainline branch back onto it; don’t merge it in.
* Don’t set up and collaborate on a separate Git server. Possibly have one to speed up clones for new developers, but don’t push anything to it that doesn’t have a `git-svn-id` entry. You may even want to add a `pre-receive` hook that checks each commit message for a `git-svn-id` and rejects pushes that contain commits without it.

If you follow those guidelines, working with a Subversion server can be more bearable. However, if it’s possible to move to a real Git server, doing so can gain your team a lot more.

## Migrating to Git ##

If you have an existing codebase in another VCS but you’ve decided to start using Git, you must migrate your project one way or another. This section goes over some importers that are included with Git for common systems and then demonstrates how to develop your own custom importer.

### Importing ###

You’ll learn how to import data from two of the bigger professionally used SCM systems — Subversion and Perforce — both because they make up the majority of users I hear of who are currently switching, and because high-quality tools for both systems are distributed with Git.

### Subversion ###

If you read the previous section about using `git svn`, you can easily use those instructions to `git svn clone` a repository; then, stop using the Subversion server, push to a new Git server, and start using that. If you want the history, you can accomplish that as quickly as you can pull the data out of the Subversion server (which may take a while).

However, the import isn’t perfect; and because it will take so long, you may as well do it right. The first problem is the author information. In Subversion, each person committing has a user on the system who is recorded in the commit information. The examples in the previous section show `schacon` in some places, such as the `blame` output and the `git svn log`. If you want to map this to better Git author data, you need a mapping from the Subversion users to the Git authors. Create a file called `users.txt` that has this mapping in a format like this:

	schacon = Scott Chacon <schacon@geemail.com>
	selse = Someo Nelse <selse@geemail.com>

To get a list of the author names that SVN uses, you can run this:

	$ svn log ^/ --xml | grep -P "^<author" | sort -u | \
	      perl -pe 's/<author>(.*?)<\/author>/$1 = /' > users.txt

That gives you the log output in XML format — you can look for the authors, create a unique list, and then strip out the XML. (Obviously this only works on a machine with `grep`, `sort`, and `perl` installed.) Then, redirect that output into your users.txt file so you can add the equivalent Git user data next to each entry.

You can provide this file to `git svn` to help it map the author data more accurately. You can also tell `git svn` not to include the metadata that Subversion normally imports, by passing `--no-metadata` to the `clone` or `init` command. This makes your `import` command look like this:

	$ git-svn clone http://my-project.googlecode.com/svn/ \
	      --authors-file=users.txt --no-metadata -s my_project

Now you should have a nicer Subversion import in your `my_project` directory. Instead of commits that look like this

	commit 37efa680e8473b615de980fa935944215428a35a
	Author: schacon <schacon@4c93b258-373f-11de-be05-5f7a86268029>
	Date:   Sun May 3 00:12:22 2009 +0000

	    fixed install - go to trunk

	    git-svn-id: https://my-project.googlecode.com/svn/trunk@94 4c93b258-373f-11de-
	    be05-5f7a86268029
they look like this:

	commit 03a8785f44c8ea5cdb0e8834b7c8e6c469be2ff2
	Author: Scott Chacon <schacon@geemail.com>
	Date:   Sun May 3 00:12:22 2009 +0000

	    fixed install - go to trunk

Not only does the Author field look a lot better, but the `git-svn-id` is no longer there, either.

You need to do a bit of `post-import` cleanup. For one thing, you should clean up the weird references that `git svn` set up. First you’ll move the tags so they’re actual tags rather than strange remote branches, and then you’ll move the rest of the branches so they’re local.

To move the tags to be proper Git tags, run

	$ cp -Rf .git/refs/remotes/tags/* .git/refs/tags/
	$ rm -Rf .git/refs/remotes/tags

This takes the references that were remote branches that started with `tag/` and makes them real (lightweight) tags.

Next, move the rest of the references under `refs/remotes` to be local branches:

	$ cp -Rf .git/refs/remotes/* .git/refs/heads/
	$ rm -Rf .git/refs/remotes

Now all the old branches are real Git branches and all the old tags are real Git tags. The last thing to do is add your new Git server as a remote and push to it. Because you want all your branches and tags to go up, you can run this:

	$ git push origin --all

All your branches and tags should be on your new Git server in a nice, clean import.

### Perforce ###

The next system you’ll look at importing from is Perforce. A Perforce importer is also distributed with Git, but only in the `contrib` section of the source code — it isn’t available by default like `git svn`. To run it, you must get the Git source code, which you can download from git.kernel.org:

	$ git clone git://git.kernel.org/pub/scm/git/git.git
	$ cd git/contrib/fast-import

In this `fast-import` directory, you should find an executable Python script named `git-p4`. You must have Python and the `p4` tool installed on your machine for this import to work. For example, you’ll import the Jam project from the Perforce Public Depot. To set up your client, you must export the P4PORT environment variable to point to the Perforce depot:

	$ export P4PORT=public.perforce.com:1666

Run the `git-p4 clone` command to import the Jam project from the Perforce server, supplying the depot and project path and the path into which you want to import the project:

	$ git-p4 clone //public/jam/src@all /opt/p4import
	Importing from //public/jam/src@all into /opt/p4import
	Reinitialized existing Git repository in /opt/p4import/.git/
	Import destination: refs/remotes/p4/master
	Importing revision 4409 (100%)

If you go to the `/opt/p4import` directory and run `git log`, you can see your imported work:

	$ git log -2
	commit 1fd4ec126171790efd2db83548b85b1bbbc07dc2
	Author: Perforce staff <support@perforce.com>
	Date:   Thu Aug 19 10:18:45 2004 -0800

	    Drop 'rc3' moniker of jam-2.5.  Folded rc2 and rc3 RELNOTES into
	    the main part of the document.  Built new tar/zip balls.

	    Only 16 months later.

	    [git-p4: depot-paths = "//public/jam/src/": change = 4409]

	commit ca8870db541a23ed867f38847eda65bf4363371d
	Author: Richard Geiger <rmg@perforce.com>
	Date:   Tue Apr 22 20:51:34 2003 -0800

	    Update derived jamgram.c

	    [git-p4: depot-paths = "//public/jam/src/": change = 3108]

You can see the `git-p4` identifier in each commit. It’s fine to keep that identifier there, in case you need to reference the Perforce change number later. However, if you’d like to remove the identifier, now is the time to do so — before you start doing work on the new repository. You can use `git filter-branch` to remove the identifier strings en masse:

	$ git filter-branch --msg-filter '
	        sed -e "/^\[git-p4:/d"
	'
	Rewrite 1fd4ec126171790efd2db83548b85b1bbbc07dc2 (123/123)
	Ref 'refs/heads/master' was rewritten

If you run `git log`, you can see that all the SHA-1 checksums for the commits have changed, but the `git-p4` strings are no longer in the commit messages:

	$ git log -2
	commit 10a16d60cffca14d454a15c6164378f4082bc5b0
	Author: Perforce staff <support@perforce.com>
	Date:   Thu Aug 19 10:18:45 2004 -0800

	    Drop 'rc3' moniker of jam-2.5.  Folded rc2 and rc3 RELNOTES into
	    the main part of the document.  Built new tar/zip balls.

	    Only 16 months later.

	commit 2b6c6db311dd76c34c66ec1c40a49405e6b527b2
	Author: Richard Geiger <rmg@perforce.com>
	Date:   Tue Apr 22 20:51:34 2003 -0800

	    Update derived jamgram.c

Your import is ready to push up to your new Git server.

### A Custom Importer ###

If your system isn’t Subversion or Perforce, you should look for an importer online — quality importers are available for CVS, Clear Case, Visual Source Safe, even a directory of archives. If none of these tools works for you, you have a rarer tool, or you otherwise need a more custom importing process, you should use `git fast-import`. This command reads simple instructions from stdin to write specific Git data. It’s much easier to create Git objects this way than to run the raw Git commands or try to write the raw objects (see Chapter 9 for more information). This way, you can write an import script that reads the necessary information out of the system you’re importing from and prints straightforward instructions to stdout. You can then run this program and pipe its output through `git fast-import`.

To quickly demonstrate, you’ll write a simple importer. Suppose you work in current, you back up your project by occasionally copying the directory into a time-stamped `back_YYYY_MM_DD` backup directory, and you want to import this into Git. Your directory structure looks like this:

	$ ls /opt/import_from
	back_2009_01_02
	back_2009_01_04
	back_2009_01_14
	back_2009_02_03
	current

In order to import a Git directory, you need to review how Git stores its data. As you may remember, Git is fundamentally a linked list of commit objects that point to a snapshot of content. All you have to do is tell `fast-import` what the content snapshots are, what commit data points to them, and the order they go in. Your strategy will be to go through the snapshots one at a time and create commits with the contents of each directory, linking each commit back to the previous one.

As you did in the "An Example Git Enforced Policy" section of Chapter 7, we’ll write this in Ruby, because it’s what I generally work with and it tends to be easy to read. You can write this example pretty easily in anything you’re familiar with — it just needs to print the appropriate information to stdout. And, if you are running on Windows, this means you'll need to take special care to not introduce carriage returns at the end your lines — git fast-import is very particular about just wanting line feeds (LF) not the carriage return line feeds (CRLF) that Windows uses.

To begin, you’ll change into the target directory and identify every subdirectory, each of which is a snapshot that you want to import as a commit. You’ll change into each subdirectory and print the commands necessary to export it. Your basic main loop looks like this:

	last_mark = nil

	# loop through the directories
	Dir.chdir(ARGV[0]) do
	  Dir.glob("*").each do |dir|
	    next if File.file?(dir)

	    # move into the target directory
	    Dir.chdir(dir) do 
	      last_mark = print_export(dir, last_mark)
	    end
	  end
	end

You run `print_export` inside each directory, which takes the manifest and mark of the previous snapshot and returns the manifest and mark of this one; that way, you can link them properly. "Mark" is the `fast-import` term for an identifier you give to a commit; as you create commits, you give each one a mark that you can use to link to it from other commits. So, the first thing to do in your `print_export` method is generate a mark from the directory name:

	mark = convert_dir_to_mark(dir)

You’ll do this by creating an array of directories and using the index value as the mark, because a mark must be an integer. Your method looks like this:

	$marks = []
	def convert_dir_to_mark(dir)
	  if !$marks.include?(dir)
	    $marks << dir
	  end
	  ($marks.index(dir) + 1).to_s
	end

Now that you have an integer representation of your commit, you need a date for the commit metadata. Because the date is expressed in the name of the directory, you’ll parse it out. The next line in your `print_export` file is

	date = convert_dir_to_date(dir)

where `convert_dir_to_date` is defined as

	def convert_dir_to_date(dir)
	  if dir == 'current'
	    return Time.now().to_i
	  else
	    dir = dir.gsub('back_', '')
	    (year, month, day) = dir.split('_')
	    return Time.local(year, month, day).to_i
	  end
	end

That returns an integer value for the date of each directory. The last piece of meta-information you need for each commit is the committer data, which you hardcode in a global variable:

	$author = 'Scott Chacon <schacon@example.com>'

Now you’re ready to begin printing out the commit data for your importer. The initial information states that you’re defining a commit object and what branch it’s on, followed by the mark you’ve generated, the committer information and commit message, and then the previous commit, if any. The code looks like this:

	# print the import information
	puts 'commit refs/heads/master'
	puts 'mark :' + mark
	puts "committer #{$author} #{date} -0700"
	export_data('imported from ' + dir)
	puts 'from :' + last_mark if last_mark

You hardcode the time zone (-0700) because doing so is easy. If you’re importing from another system, you must specify the time zone as an offset. 
The commit message must be expressed in a special format:

	data (size)\n(contents)

The format consists of the word data, the size of the data to be read, a newline, and finally the data. Because you need to use the same format to specify the file contents later, you create a helper method, `export_data`:

	def export_data(string)
	  print "data #{string.size}\n#{string}"
	end

All that’s left is to specify the file contents for each snapshot. This is easy, because you have each one in a directory — you can print out the `deleteall` command followed by the contents of each file in the directory. Git will then record each snapshot appropriately:

	puts 'deleteall'
	Dir.glob("**/*").each do |file|
	  next if !File.file?(file)
	  inline_data(file)
	end

Note:	Because many systems think of their revisions as changes from one commit to another, fast-import can also take commands with each commit to specify which files have been added, removed, or modified and what the new contents are. You could calculate the differences between snapshots and provide only this data, but doing so is more complex — you may as well give Git all the data and let it figure it out. If this is better suited to your data, check the `fast-import` man page for details about how to provide your data in this manner.

The format for listing the new file contents or specifying a modified file with the new contents is as follows:

	M 644 inline path/to/file
	data (size)
	(file contents)

Here, 644 is the mode (if you have executable files, you need to detect and specify 755 instead), and inline says you’ll list the contents immediately after this line. Your `inline_data` method looks like this:

	def inline_data(file, code = 'M', mode = '644')
	  content = File.read(file)
	  puts "#{code} #{mode} inline #{file}"
	  export_data(content)
	end

You reuse the `export_data` method you defined earlier, because it’s the same as the way you specified your commit message data. 

The last thing you need to do is to return the current mark so it can be passed to the next iteration:

	return mark

NOTE: If you are running on Windows you'll need to make sure that you add one extra step. As mentioned before, Windows uses CRLF for new line characters while git fast-import expects only LF. To get around this problem and make git fast-import happy, you need to tell ruby to use LF instead of CRLF:

	$stdout.binmode

That’s it. If you run this script, you’ll get content that looks something like this:

	$ ruby import.rb /opt/import_from 
	commit refs/heads/master
	mark :1
	committer Scott Chacon <schacon@geemail.com> 1230883200 -0700
	data 29
	imported from back_2009_01_02deleteall
	M 644 inline file.rb
	data 12
	version two
	commit refs/heads/master
	mark :2
	committer Scott Chacon <schacon@geemail.com> 1231056000 -0700
	data 29
	imported from back_2009_01_04from :1
	deleteall
	M 644 inline file.rb
	data 14
	version three
	M 644 inline new.rb
	data 16
	new version one
	(...)

Para ejecutar el importador, canaliza esta salida a través de `git fast-import` mientras esté en el directorio Git al que desee importar. Puede crear un nuevo directorio y luego ejecutar `git init` en él para un punto de partida, y luego ejecutar su script:

	$ git init
	Initialized empty Git repository in /opt/import_to/.git/
	$ ruby import.rb /opt/import_from | git fast-import
	git-fast-import statistics:
	---------------------------------------------------------------------
	Alloc'd objects:       5000
	Total objects:           18 (         1 duplicates                  )
	      blobs  :            7 (         1 duplicates          0 deltas)
	      trees  :            6 (         0 duplicates          1 deltas)
	      commits:            5 (         0 duplicates          0 deltas)
	      tags   :            0 (         0 duplicates          0 deltas)
	Total branches:           1 (         1 loads     )
	      marks:           1024 (         5 unique    )
	      atoms:              3
	Memory total:          2255 KiB
	       pools:          2098 KiB
	     objects:           156 KiB
	---------------------------------------------------------------------
	pack_report: getpagesize()            =       4096
	pack_report: core.packedGitWindowSize =   33554432
	pack_report: core.packedGitLimit      =  268435456
	pack_report: pack_used_ctr            =          9
	pack_report: pack_mmap_calls          =          5
	pack_report: pack_open_windows        =          1 /          1
	pack_report: pack_mapped              =       1356 /       1356
	---------------------------------------------------------------------

Como puede ver, cuando se completa con éxito, le da un montón de estadísticas sobre lo que logró. En este caso, importó 18 objetos en total para 5 confirmaciones en 1 rama. Ahora, puede ejecutar `git log` para ver su nuevo historial:

	$ git log -2
	commit 10bfe7d22ce15ee25b60a824c8982157ca593d41
	Author: Scott Chacon <schacon@example.com>
	Date:   Sun May 3 12:57:39 2009 -0700

	    imported from current

	commit 7e519590de754d079dd73b44d695a42c9d2df452
	Author: Scott Chacon <schacon@example.com>
	Date:   Tue Feb 3 01:00:00 2009 -0700

	    imported from back_2009_02_03

Ahí tienes, un repositorio Git limpio y bonito. Es importante tener en cuenta que no hay nada extraído - usted no tiene ningún archivo en su directorio de trabajo en un primer momento. Para obtenerlas, debe restablecer su sucursal a donde `master` es ahora:

	$ ls
	$ git reset --hard master
	HEAD is now at 10bfe7d imported from current
	$ ls
	file.rb  lib

Puede hacer mucho más con la herramienta `fast-import` - manejar diferentes modos, datos binarios, múltiples ramas y combinaciones, etiquetas, indicadores de progreso y más. Hay varios ejemplos de escenarios más complejos disponibles en el directorio `contrib / fast-import` del código fuente de Git; Uno de los mejores es el guión `git-p4` que acabo de cubrir.

## Recapitulando ##

Debería sentirse cómodo usando Git con Subversion o importando casi cualquier repositorio existente en un nuevo Git uno sin perder datos. El siguiente capítulo cubrirá los elementos internos de Git para que pueda crear cada byte, si es necesario.
