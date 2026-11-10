# Laboratorio virual para monitoreo con Wazuh

## Objetivo
Utilizar máquinas virtuales para generar un espacio que permita simular la implementación, exploración y análisis con una plataforma SIEM (Wazuh).

## Topología
Se utilizó un Hipervisor KVM/QEMU a través de libvirt.  
Las máquinas utilizadas fueron:  
* Ubuntu Server con 1.5 GB de RAM y 1 núcleo de procesador (máquina víctima y agente)
* Ubuntu server con 4 GB de RAM y 2 núcleos, con Wazuh (SIEM)
* Linux Mint (computadora atacante y máquina principal) 8 GB RAM + 2 GB sway - memoria asignada a máquinas virtuales.

## Instalar Ubuntu server. 
Si se intenta instalar directo de la imagen ISO descargada de la página web, puede fallar. Por eso, hay que simular una unidad de arranque (el archivo con la imagen ISO) y crear un "disco vacío virtual" a través del espacio details de la máquina virtual (para libvirt), y utilizando las funciones añadir hardware y boot menu, demanera similar a como se haría con una computadora física.

## Crear la máquina con Wazuh

**Paso 1: Conectar a través de SSH (Opcional)**  
La forma más sencilla de instalar todo es a través de SSH, para poder copiar y pegar links desde los sitios oficiales en la computadora principal.  
En la computadora física se abre una terminal y se ingresa el comando:  
`ssh <user>@<ip_máquina>`  
(Reemplazando el usuario y la IP)  
* Si no se conoce la IP, se puede obtener a través del comando `ip a`

**Paso 2: Instalar Wazuh**  
Se instalan las dependencias de Wazuh:
`curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a`
Si los requisitos del sistema son insuficientes, hay que evadir con -i. Si se ha realizado antes algún proceso y ha fallado, hay que forzar a sobreescribir con -o (duda)  

Algunos problemas con los que es posible encontrarse al instalar Wazuh son los siguientes:  
1. Falta de RAM: Como este proyecto está pensado para una RAM muy limitada, se puede configurar RAM virtual (tanto en la PC original como en el Server) para evitar que los procesos se detengan o bloqueen.  
Primero se crea el archivo swap, luego se asignan permisos (solo administrador), se formatea y se activa.  
`sudo fallocate -l 4G /swapfile`  
`sudo chmod 600 /swapfile`  
`sudo mkswap /swapfile`  
`sudo swapon /swapfile`  
2. Es posible que esto llene el disco si no hay suficiente espacio, este archivo puede saturar el disco y se bloqueó para solo lectura. Para liberar el archivo de memoria y utilizarlo para el disco, usar:  
`sudo swapoff /swapfile`  
`sudo rm /swapfile`  
`sudo sed -i '/swapfile/d' /etc/fstab`  
3. El tamaño del disco tiene que ser de al menos 40 GB, pero es posible que el Ubuntu Server bloquee cierto espacio por cuestiones de seguridad. Hay que desbloquearlo. 
Para resolverlo, primero hay que analizar el espacio en el disco: df -h
Se le dice a Ubuntu que tome el 100% de espacio: `sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`  
Y se redimensiona el sistema de archivos: `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`
Con `df -h` nos aseguramos de que el espacio sea suficiente.  
4. Si el disco se satura, es posible que se bloquee por seguridad para evitar sobrecargar al sistema. Ya que se ha corregido el espacio del disco, se puede utilizar:  
`sudo curl -k -X PUT "https://localhost:9200/_all/_settings" \`  
`--cert /etc/wazuh-indexer/certs/admin.pem \`  
`--key /etc/wazuh-indexer/certs/admin-key.pem \`  
`-H "Content-Type: application/json" \`  
`-d '{"index.blocks.read_only_allow_delete": null}'`  
Y luego reiniciar los servicios para que se apliquen los cambios:  
`sudo systemctl restart wazuh-indexer wazuh-manager wazuh-dashboard`  


Hay que convertir la dirección IP en una estática para evitar problemas con los agentes en el futuro. Para conocer la información de red de forma simple se usó:  
`ip route | grep default`  
Que imprimió:  
`default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.179 metric 100`  
Y para saber el nombre del archivo de configuración de red:  
`ls /etc/netplan/`  
Que arrojó:  
`00-installer-config.yaml`  

Se modifica (notar que se usó el nombre que arrojó el ls, este nombre puede variar):  
`nano /etc/netplan/00-installer-config.yaml`  

Y se edita, teniendo mucho cuidado con los espacios, notar los nombres de la interfaz, la default gateway y la IPv4 que conocimos con ip a (para evitar que ssh se desconecte):  
```network:
  ethernets:
    enp1s0:
      dhcp4: false
      addresses:
        - 192.168.122.179/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      dhcp6: true
      match:
        macaddress: 52:54:00:97:d7:bf
      set-name: enp1s0
  version: 2
```
Guardar y cerrar (CTRL+O, enter, CTRL+X) y decirle a Ubuntu que lea esta nueva configuración:  
`sudo netplan apply`  

**Paso 3: Instalar un primer agente**  
Se puede abrir Wazuh desde una computadora en la misma red (recomendable en la máquina principal), poniendo en el navegador https://[ip_SIEMServer]  
(Por ejemplo: [https://192.168.122.179](https://192.168.122.179))  
1. Ingresar el usuario y la contraseña.  
2. Una vez abierta la interfaz, dar clic en Deploy new agent  
3. Wazuh muestra una interfaz donde indica los pasos a seguir según las características del sistema en el que se quiera agregar al agente. Este es un ejemplo:  
`wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb && sudo WAZUH_MANAGER='192.168.122.179' WAZUH_AGENT_NAME='agent_ex' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb`  
4. Iniciar el agente, usando (también indicadas desde la interfaz de Wazuh):  
`sudo systemctl daemon-reload`  
`sudo systemctl enable wazuh-agent`  
`sudo systemctl start wazuh-agent`  
  
**Paso 4: Simular ataques**  
Es posible conectarse a la máquina víctima desde otra máquina virtual (ejemplo, Kali Linux), pero para evitar sobrecargar la RAM de la computadora principal, se puede utilizar a esta misma como atacante.   
1. Conectarse a la máquina víctima (lo más simple es usar `ssh usuario@IP_Victima`) y cometer varios errores al ingresar la contraseña.   
2. Utilizar hydra para lanzar ataques de fuerza bruta (con un diccionario preexistente):  
`hydra -l [usuario_victima] -P diccionario.txt ssh://[IP_victima]`  
3. Realizar ataques de reconocimiento (`nmap -A -T4 IP_victima`)  
4. Observar las evidencias de los ataques en Wazuh.  
<center><img src="images/graphsprat1.png" width = "700"></center>  
Los ataques de hydra son evidentes a través de la leyenda "password guessing", mientras los intentos de autenticación con SSH corresponden al resto de la actividad. Dado que el escaneo de Nmap (-A) solo verifica versiones de protocolo sin intentar un inicio de sesión completo, su actividad no detona alertas mayores en el dashboard de MITRE ATT&CK, pero sí deja rastro de errores de conexión (protocol identification) en los logs crudos (Events).  

## Anexo: Recuperación de acceso físico
En caso de olvidar/perder los datos de las máquinas virtuales, es posible recurrir a una característica específica de los sistemas Linux.  
Los sistemas Linux sin protecciones se pueden iniciar sin usuario y contraseña fácilmente:  
1. Reiniciar o apagar y encender. En cuanto comience a encender, ingresar al menú GRUB, presionando shift o ESC repetidas veces.  
2. Una vez abierto el menú, seleccionar el sistema operativo (que para este caso fue ubuntu), y cuando esté ahí, presionar e para abrir un editor de texto con configuración de arranque.  
3. En la línea que empieza con la palabra linux, ir hasta el final de esa línea usando las flechas. Al final de esa línea, escribir un espacio y agregar:  
`rw init=/bin/bash`  
Una vez escrito, presionar Ctrl + X (o F10) para arrancar el sistema con esta configuración.
4. para ver el nombre de usuario, ingresar `ls /home`  
5. Cambiar la contraseña con un comando passwd <usuario> e ingresar la contraseña cuando la pida.  
6. Forzar un inicio desde la terminal `exec /sbin/init`  

Hay algunas maneras de protegerlo, como cifrar el disco, ponerle una contraseña a la BIOS/UEFI y/o ponerle contraseña al menú GRUB.  
