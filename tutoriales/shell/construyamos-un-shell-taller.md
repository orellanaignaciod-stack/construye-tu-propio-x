# ¡Construyamos un shell! — Taller práctico

*Kamal Marhubi — Strange Loop 2014*  
*Traducción al español: [orellanaignaciod-stack](https://github.com/orellanaignaciod-stack) — bajo licencia [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es)*  
*Repositorio original (en inglés): [github.com/kamalmarhubi/shell-workshop](https://github.com/kamalmarhubi/shell-workshop)*

---

## Introducción

La mayoría de nosotros usa un shell, al menos de vez en cuando. Algunos pasamos la mayor parte de nuestro tiempo en uno. Con frecuencia lo damos por sentado, sin saber ni importarnos cómo hace lo que hace. En este taller vamos a profundizar un poco en eso, ¡construyendo nuestro propio shell desde cero!

### Enfoque del taller

Los shells tienen muchas funciones. Este taller se va a enfocar en cómo se lanzan procesos y cómo se puede controlar su entrada/salida (IO) con pipes. Dejaremos fuera cosas como la expansión de variables de entorno, el manejo de señales y otras, pero al final veremos algunos ejemplos de otras direcciones posibles.

**Regla 0.** En particular, no nos interesa lidiar con un montón de *detalles* complicados, incluyendo pero no limitado a:

- Gestión de memoria
- Parseo, tokenización y procesamiento de texto
- Manejo "correcto" de errores

---

## 0. Empezando

Descarga el código esqueleto del taller y las funciones utilitarias desde el [repositorio original](https://github.com/kamalmarhubi/shell-workshop).

Se recomienda **fuertemente** usar control de versiones y hacer commits frecuentes.

---

## 1. Dale nombre a tu shell y lee la entrada

Lo ***más importante*** para empezar es elegir un nombre para tu shell y su prompt. Kamal usó `heeee>` mientras armaba el taller, así que una interacción típica se ve así:

```
heeee> ls
shell   shell.c
```

¡Manos a la obra!

**Ejercicio 1:** Configura la función `main` de tu shell para que pida al usuario una línea de texto repetidamente. Por ahora, solo vamos a imprimir esa línea de vuelta. ¡Nuestro shell es un `cat` ligeramente glorificado, pero más personalizado!

Este fragmento lee una línea en C en la variable `line`:

```c
char *line = NULL;
int capacity = 0;
getline(&line, &capacity, stdin);
```

---

## 2. Llamadas al sistema

Ya podemos leer el comando del usuario. El siguiente paso es ejecutarlo. Para eso necesitamos ayuda del kernel, que se encarga de casi todo lo relacionado con los procesos, entre muchas otras cosas.

La forma en que nuestros programas hablan con el kernel es a través de las **llamadas al sistema** (*system calls*). Son operaciones que le pedimos al kernel que haga por nosotros. Algunos ejemplos:

- Abrir archivos (`open`)
- Leer y escribir en ellos (`read`, `write`)
- Enviar y recibir datos por la red (`sendto`, `recvfrom`)
- Iniciar programas (`execve`)

En la práctica, casi todo lo que involucra que nuestros programas interactúen con el "mundo real" requiere llamadas al sistema.

En este momento nos interesa ejecutar un programa. Para eso se usa la llamada al sistema `execve`, y nosotros usaremos una función wrapper llamada `execvp`.

---

## 3. Ejecutar un programa

Antes de correr un programa, necesitamos entender cómo se pasan los argumentos.

Cuando se ejecuta un programa, recibe un array de argumentos llamado `argv`. Por convención, el primero es el nombre del programa. Entonces, cuando corremos algo como `ls -l /tmp`, el `argv` que recibe `ls` es `{ "ls", "-l", "/tmp" }`.

La llamada sería `execvp('ls', ['ls', '-l', '/tmp'])`, aunque C no te deja escribir el array así directamente.

Ya estamos listos para correr un programa. Parsear `ls -l /tmp` en un array de argumentos es tedioso en C (y poco instructivo), así que hay una función `parse_command` disponible en el código del taller.

**Ejercicio 2:** Modifica tu shell para que, después de leer una línea desde `stdin`, ejecute el comando con sus argumentos.

Pruébalo. ¿Ocurrió algo inesperado?

---

## 4. Crear un proceso nuevo con fork

En el ejercicio anterior, una vez que el programa terminó, no volviste a tu prompt sino que fuiste directamente de vuelta a tu shell habitual. La llamada `exec` *transforma* el proceso actual en el comando que especificamos. Así que el proceso de nuestro shell es *reemplazado* por `ls` o lo que hayamos elegido. Una vez que ese programa termina, el proceso sale — no hay más shell.

Para evitar esto, queremos que el comando corra como un nuevo proceso. Los procesos se crean con la llamada al sistema `fork`. Esta crea un nuevo proceso que es un clon del existente, incluyendo el punto del programa que se está ejecutando. Luego podemos llamar a `exec` en el proceso hijo recién creado, dejando intacto el proceso de nuestro shell para futuras interacciones.

El valor de retorno de `fork` nos dice si estamos en el hijo o en el padre: retorna `0` para el hijo y el ID del nuevo proceso en el padre.

**Ejercicio 3:** Modifica tu shell para llamar a `fork`, y luego a `exec` dentro del hijo.

¡Felicitaciones! Tu shell ya es levemente útil. Todavía hay un problema: cuando corres un comando, probablemente el prompt se imprime antes que la salida del comando.

---

## 5. Esperar a los hijos

Si miras tu código, verás que el padre vuelve inmediatamente al prompt. Para esperar al proceso hijo, existe la llamada al sistema `wait`. Para nuestros propósitos, podemos llamarla con argumento `NULL`: `wait(NULL);`.

**Ejercicio 4:** Haz que tu shell use `wait` para esperar al proceso hijo antes de volver al prompt.

¡Ahora empieza a parecerse a un shell de verdad! Prueba algunos de tus comandos favoritos.

---

## 5. Comandos internos (Builtins)

¿Qué pasa si intentamos correr `cd`? Existe un programa en `/bin/cd` que debería permitirnos cambiar de directorio. Prueba esta secuencia:

```
heeee> pwd
/Users/kalmar/projects/conferences/2014/strangeloop/shell-workshop
heeee> cd ..
heeee> pwd
[¿algo inesperado?]
heeee> cd /tmp
heeee> pwd
[¿algo inesperado?]
```

Para entender por qué no podemos cambiar de directorio, necesitamos saber un poco más sobre cómo funcionan los procesos.

Cada proceso tiene:

- Su propia memoria (donde viven las variables globales, locales, el stack y el heap)
- Metadatos adicionales que gestiona el kernel, pero que el proceso no puede cambiar directamente. Puede cambiarlos mediante llamadas al sistema:
  - ID de usuario (`setuid`)
  - ID de grupo (`setgid`)
  - Prioridad (`setpriority`)
  - **Directorio de trabajo** (`chdir`) ← este es el que nos importa

Imaginemos que hacemos `fork` y `exec` con el proceso `/bin/cd`:

```
1) padre: pid 1000, directorio /tmp
---- fork ----
2) padre: pid 1000, directorio /tmp
   hijo:  pid 2000, directorio /tmp
---- exec `/bin/cd /home/awesome` en el hijo ----
3) padre: pid 1000, directorio /tmp
   hijo:  pid 2000, directorio /tmp/awesome
---- el hijo termina ----
4) padre: pid 1000, directorio /tmp   ← el padre no cambió
```

Un proceso solo puede modificar su propio estado con llamadas al sistema. Por eso el shell padre sigue con el mismo directorio. Para que `cd` funcione, el propio shell tiene que llamar a `chdir`. Eso hace que `cd` sea un **builtin** — un comando integrado al shell.

**Ejercicio 6:** Agrega un builtin `cd` a tu shell.

`cd` no es el único builtin de los shells habituales. Puedes ver más con `man builtin`. Nota que no todos *necesitan* ser builtins — por ejemplo `echo` funciona bien como comando externo, pero muchos shells lo incluyen como builtin por eficiencia, ya que crear un nuevo proceso tiene cierto costo.

---

## 6. Pipelines

A continuación vamos a implementar pipelines. Prueba correr `ls | wc` en tu shell. ¿Qué pasa?

```
heeee> ls | wc
ls: cannot access |: No such file or directory
ls: cannot access wc: No such file or directory
```

Ahora mismo no sabe que `|` tiene algún significado especial, e intenta listar los archivos `|` y `wc`. Antes de arreglarlo, necesitamos entender mejor cómo funcionan los pipes.

Los archivos en sistemas Unix se gestionan usando **descriptores de archivo** (*file descriptors*). En Linux puedes ver todos los descriptores de un proceso con `ls -l /proc/<pid>/fd`. Son enteros positivos: `0` es stdin, `1` es stdout, `2` es stderr, y del 3 en adelante cualquier otro archivo que el proceso tenga abierto.

La llamada al sistema `pipe` crea dos nuevos descriptores con una propiedad especial:

```c
int pipe_fds[2];
pipe(pipe_fds);  // retorna algo como [4, 5] en pipe_fds
```

Si `pipe` retorna `[4, 5]`, entonces cualquier dato escrito en `5` puede leerse desde `4`. Esto significa que necesitamos que `ls` escriba en `5` (su stdout) y que `wc` lea desde `4` (su stdin). La forma de cambiar a qué archivo apunta un descriptor es con la llamada `dup2`. En este ejemplo haríamos `dup2(5, 1)` para `ls` y `dup2(4, 0)` para `wc`.

**En resumen**, para crear el pipe `ls | wc`:

1. Crear un pipe con `pipe`. Supongamos que devuelve los descriptores `4` y `5`.
2. Hacer dos `fork` (uno para `ls` y otro para `wc`).

```
padre:  pid 1000, stdin: terminal, stdout: terminal
---- fork dos veces ----
padre:  pid 1000, stdin: terminal, stdout: terminal
hijo 1: pid 2000, stdin: terminal, stdout: terminal  (para `ls`)
hijo 2: pid 3000, stdin: terminal, stdout: terminal  (para `wc`)
```

3. En hijo 1: ejecutar `dup2(pipe_fds[1], 1)` y hacer exec de `ls`
4. En hijo 2: ejecutar `dup2(pipe_fds[0], 0)` y hacer exec de `wc`
5. En el padre: cerrar ambos extremos del pipe.

¡Eso es todo!

**Ejercicio 7:** Escribe una función `fork_and_exec_with_io(cmd* cmd, int stdout_fd, int stdin_fd)` que haga fork y exec del comando especificado, y cambie su stdin y stdout según lo indicado.

**Ejercicio 8:** Usa `fork_and_exec_with_io` para implementar pipes de 2 elementos.

---

## 7. ¿Qué sigue?

Hay varias direcciones posibles una vez que completes los ejercicios. Las dividimos en progresiones naturales y otras áreas de exploración.

### Progresiones naturales

#### 1. Pipelines más largos
Implementamos un solo pipe, pero los más largos son mucho más útiles. La función `parse_line` puede retornar cualquier número de comandos encadenados — ¡depende de ti hacerlo funcionar!

#### 2. Redirección de IO hacia/desde archivos
Si corres `grep shell /usr/share/dict/words > /tmp/resultados` en tu shell habitual, la lista termina en `/tmp/resultados`. Para implementarlo, necesitarás extender el código de parseo y usar `open` en lugar de `pipe` para obtener el descriptor.

#### 3. Redirigir otros descriptores de archivo
Si corres `algún-comando 2>/tmp/errores`, estás redirigiendo `stderr` (fd 2) al archivo. Es una generalización de lo anterior.

#### 4. Redirigir *hacia* otros descriptores
Con `algún-comando 2>&1`, los datos escritos en `stderr` van al mismo lugar que `stdout`. Es una extensión más de la redirección.

### Otras direcciones

#### 1. Expansión de variables de entorno
Compara la salida de `echo $HOME` en tu shell vs. el habitual. Los shells reemplazan los tokens que empiezan con `$` por el valor de la variable de entorno correspondiente. La función `getenv` busca una variable en el entorno.

#### 2. Asignar variables de entorno a procesos hijos
Compara `date` y `TZ=Pacific/Samoa date`. El prefijo `VAR=valor` antes de un comando asigna esa variable en el entorno del hijo. La función `setenv` permite modificar variables de entorno.

#### 3. Otro builtin: `export`
En bash, `export` permite cambiar el valor de una variable de entorno en el propio shell. Como con `cd`, debe ser un builtin porque un hijo no puede modificar el entorno de su padre.

#### 4. Manejo de señales
Compara qué pasa cuando presionas `^C` (Control-C) en tu shell vs. el habitual. `^C` envía la señal `SIGINT` al proceso. Por defecto esto lo termina. Puedes registrar una función manejadora de señal para evitarlo.

#### 5. Globbing
Compara `wc -c *` en tu shell vs. el habitual. Expandir el comodín `*` lo hace el shell antes de pasar los argumentos al hijo. La función `glob` del header `<glob.h>` será útil.

#### 6. Correr un comando en segundo plano
En el shell habitual, terminar un comando con `&` lo corre en segundo plano. El comando corre pero el shell no se bloquea esperándolo — es decir, no hace `wait` en el proceso hijo.

#### 7. Control de trabajos (job control)
Una vez implementado el segundo plano, puedes agregar `fg`, `bg` y `jobs` para manejar los procesos en segundo plano como lo hace bash o zsh.

---

*Taller original de Kamal Marhubi, presentado en Strange Loop 2014. Repositorio: [github.com/kamalmarhubi/shell-workshop](https://github.com/kamalmarhubi/shell-workshop). Traducido y adaptado al español bajo los términos de la licencia [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es).*
