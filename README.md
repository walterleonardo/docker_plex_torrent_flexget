# Utilizaremos un repo base, de un proyecto creado por PELADO NERDS, pero con una modifiación, utilizando la imagen plex original compilada para la plataforma ARM armv7 para 32bits y arm64 para 64bits

docker build -t plexinc/pms-docker:latest -f Dockerfile.armv7 . # or arm64

Si no funciona utilicen el Buildx
sudo docker buildx build -t plexinc/pms-docker:latest --platform linux/arm64 .


# Plex sobre Docker en Raspberry

Con este repo podes crear tu propio server que descarga tus series y peliculas automáticamente, y cuando finaliza, las copia al directorio `media/` donde Plex las encuentra y las agrega a tu biblioteca.

También agregué un pequeño server samba por si querés compartir los archivos por red

## Requerimientos iniciales
Normalmente linux ya crea una variable llamada $USER, con el nombre del $USER logeado...
por eso utilizaremos esta variable en todos los comandos.


```
sudo useradd $USER -G sudo
```

Agregar esto al sudoers para correr sudo sin password

```
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

Agregar esta linea a `sshd_config` para que sólo tu  usuario pueda hacer ssh

```
echo "AllowUsers $USER" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl enable ssh && sudo systemctl start ssh
```

Instalar paquetes básicos, ntfs-3g es solo si quieres montar un disco formateado con NTFS

```
sudo apt-get update && sudo apt-get install -y \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common \
     vim \
     fail2ban \
     ntfs-3g
```

Instalar Docker

```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
echo "deb [arch=armhf] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y --no-install-recommends docker-ce docker-compose
```
Si no funciona o da error probar directamente con:
'''
sudo apt install docker
'''
Ya las versiones y repositorios incluyen el docker en sus listas.

Modificá tu docker config para que guarde los temps/imagenes en el disco:

```
sudo vim /etc/default/docker
# Agregar esta linea al final con la ruta de tu disco externo montado
export DOCKER_TMPDIR="/mnt/storage/docker-tmp"
```

Agregar tu usuario al grupo docker 

```
sudo usermod -a -G docker $USER
#(logout and login)
```
Si quieres montar un disco para tener mas espacio necesitas hacer estos pasos.
Te recomiendo formatearlo en EXT4, ya que cualquier otro tipo de formato puede tener problemas a la hora de cambiar OWNERS o PERMISOS:
Para montar el disco NTFS es necesario ntfs-3g
```
# usamos la terminal como root porque vamos a ejecutar algunos comandos que necesitan ese modo de ejecución
sudo su
# conectamos por USB y buscamos el disco que querramos montar (por ejemplo la partición sdb1 del disco sdb)
fdisk -l
# pueden usar el siguiente comando para obtener el UUID
ls -l /dev/disk/by-uuid/
# y simplemente montamos el disco en el archivo /etc/fstab (pueden hacerlo por el editor que les guste o por consola)
echo UUID=XXXXXX-XXXX-XXX-XXX /mnt/storage ext4 defaults,auto,users,rw,nofail 0 0 | sudo tee -a /etc/fstab
# por último para que lea el archivo fstab
mount -a (o reiniciar)
```

## Deberemos compilar el flex para nuestra plataforma... linux/armv7 (32bits) o linux/arm64 (64bits), en nuestro caso como tenemos un raspberry pi 3 es de 64 bits.
'''
sudo docker build -t plexinc/pms-docker:latest -f Dockerfile.arm64 .
'''

## Cómo correrlo

Simplemente bajate este repo y modificá las rutas de tus archivos en el archivo (oculto) .env, y después corré:

`docker-compose up -d`

## IMPORTANTE

Tenemos que crear el directorio a mano. Para que inicie el flexget
/mnt/storage/torrents $ sudo mkdir complete


Las raspberry son computadoras excelentes pero no muy potentes, y plex por defecto intenta transcodear los videos para ahorrar ancho de banda (en mi opinión, una HORRIBLE idea), y la chiquita raspberry no se aguanta este transcodeo "al vuelo", entonces hay que configurar los CLIENTES de plex (si, hay que hacerlo en cada cliente) para que intente reproducir el video en la máxima calidad posible, evitando transcodear y pasando el video derecho a tu tele o Chromecast sin procesar nada, de esta forma, yo he tenido 3 reproducciones concurrentes sin ningún problema. En android y iphone las opciones son muy similares.

# info adicional y util.
Para montar un disco remoto con samba.
'''
sudo mount -t cifs -o username=walterleonardo //10.38.70.47/Public /mnt/temp/
'''
