# Despliegue con Ansible: LAMP, Samba, FTP y PostgreSQL

Proyecto de Ansible (con roles) para desplegar de forma remota, sobre **openSUSE Leap 16.0**, los siguientes servicios:

| Rol | Qué instala y configura |
|---|---|
| `mariadb` | MariaDB y contraseña de `root` (idempotente) |
| `lamp` | Apache, PHP 8 y phpMyAdmin (publicado en `/phpMyAdmin`) |
| `samba` | Usuario de sistema, usuario Samba, directorio compartido y `create mask = 0664` |
| `ftp` | vsftpd con modo pasivo (puertos 30000-30100) |
| `postgresql` | PostgreSQL con SSL, acceso remoto, contraseña de `postgres`, usuario y base de datos nuevos, y librería PHP (`php8-pgsql`) |

Las contraseñas **no están en texto plano**: se guardan cifradas con Ansible Vault, y la conexión usa llaves SSH.

---

## 1. Arquitectura

Se usan dos máquinas:

- **Nodo de control**: la máquina donde se descarga este repositorio con `git` y desde donde se ejecuta Ansible.
- **Servidor destino**: la máquina openSUSE Leap 16.0 donde se instalan los servicios. No necesita tener el repositorio ni Ansible, solo SSH.

```
[ Nodo de control ]  --SSH-->  [ Servidor destino (openSUSE 16.0) ]
  git + ansible                  Apache, MariaDB, Samba, vsftpd, PostgreSQL
```

---

## 2. Requisitos

### Nodo de control
- openSUSE (o cualquier Linux) con acceso a internet.
- Paquetes: `git`, `ansible` y `python3-passlib`.
- Acceso de red al servidor destino (puerto 22).
- Una cuenta de GitHub con acceso al repositorio (si es privado, ver el paso 3).

### Servidor destino
- openSUSE Leap 16.0 con acceso a internet (repositorios de zypper).
- Servicio SSH activo (`sshd`).
- Un usuario con permisos `sudo` (en este proyecto: `publica`).
- Python 3 instalado (viene por defecto en openSUSE).
- `firewalld` activo (los roles abren puertos con él).

### Puertos que abre el despliegue
| Servicio | Puerto |
|---|---|
| HTTP (Apache, phpMyAdmin) | 80 |
| Samba | 139, 445 |
| FTP | 21 y 30000-30100 (modo pasivo) |
| PostgreSQL | 5432 |

---

## 3. Instalación paso a paso (desde GitHub)

### 3.1 Instalar las herramientas en el nodo de control

```bash
sudo zypper install git ansible python3-passlib
```

### 3.2 Descargar el repositorio con git

**Si el repositorio es público:**

```bash
git clone https://github.com/dagda-1/ansible-lamp-samba-ftp.git
cd ansible-lamp-samba-ftp
```

**Si el repositorio es privado**, hay que autenticarse. La forma recomendada es con una llave SSH:

```bash
# 1. Crear una llave para GitHub (Enter en todas las preguntas)
ssh-keygen -t ed25519 -f ~/.ssh/id_github -C "nodo-control"

# 2. Decirle a SSH que la use con GitHub
cat >> ~/.ssh/config <<'EOF'
Host github.com
    IdentityFile ~/.ssh/id_github
EOF
chmod 600 ~/.ssh/config

# 3. Mostrar la llave pública
cat ~/.ssh/id_github.pub
```

Copia esa línea en GitHub: **Settings → SSH and GPG keys → New SSH key**. Luego comprueba y clona:

```bash
ssh -T git@github.com        # debe decir: Hi USUARIO! You've successfully authenticated...
git clone git@github.com:dagda-1/ansible-lamp-samba-ftp.git
cd ansible-lamp-samba-ftp
```

> Tu cuenta de GitHub debe tener acceso al repositorio (el dueño debe haberte invitado como colaborador).

### 3.3 Instalar las colecciones de Ansible

```bash
ansible-galaxy collection install -r requirements.yml
```

Instala `community.general`, `community.mysql`, `community.postgresql` y `ansible.posix`.

### 3.4 Preparar el acceso SSH al servidor destino

Para no usar contraseñas visibles, el nodo de control entra con llave SSH.

En el **servidor destino**, asegúrate de que SSH esté activo:

```bash
sudo systemctl enable --now sshd
```

En el **nodo de control**, crea una llave (si no tienes una) y cópiala al servidor:

```bash
ssh-keygen -t ed25519            # Enter en todas las preguntas
ssh-copy-id USUARIO@IP_DEL_SERVIDOR
ssh USUARIO@IP_DEL_SERVIDOR      # debe entrar SIN pedir contraseña
```

Reemplaza `USUARIO` e `IP_DEL_SERVIDOR` por los datos reales (por ejemplo `publica` y `192.168.1.50`).

---

## 4. Configuración para tu entorno

### 4.1 Indicar la IP del servidor (`inventory/hosts.yml`)

```yaml
all:
  children:
    servers:
      hosts:
        opensuse01:
          ansible_host: IP_DEL_SERVIDOR
```

El archivo solo tiene la IP: no contiene usuario ni contraseña.

### 4.2 Indicar el usuario de conexión (`inventory/group_vars/all/vars.yml`)

Las primeras líneas definen cómo se conecta Ansible:

```yaml
ansible_user: publica
ansible_become: true
ansible_become_password: "{{ vault_become_password }}"
```

- `ansible_user`: el usuario del servidor destino (el mismo del `ssh-copy-id`).
- `ansible_become_password`: contraseña de `sudo` de ese usuario, que se toma del vault.

Más abajo en el mismo archivo puedes cambiar, si quieres, los nombres de los usuarios y recursos:

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `samba_user` | Usuario de sistema y de Samba | `sambauser` |
| `samba_share_path` | Directorio compartido | `/srv/samba/compartido` |
| `pg_app_user` | Usuario nuevo de PostgreSQL | `appuser` |
| `pg_app_db` | Base de datos del usuario nuevo | `appdb` |

Variable opcional del role `postgresql` (en `roles/postgresql/defaults/main.yml`): `pg_remote_network`, que define quién puede conectarse remotamente. Por defecto es `0.0.0.0/0` (cualquier IP, siempre con SSL). Para mayor seguridad, límitalo a tu red, por ejemplo `192.168.1.0/24`.

### 4.3 Crear tu propio vault de contraseñas

El `vault.yml` del repositorio está cifrado con una contraseña que **solo conoce su autor**, así que en tu entorno debes crear el tuyo.

**1. Borra el vault existente y crea uno nuevo:**

```bash
rm inventory/group_vars/all/vault.yml
EDITOR=nano ansible-vault create inventory/group_vars/all/vault.yml
```

Te pedirá una contraseña del vault (dos veces). Elige una larga y guárdala. Se abrirá un editor: escribe estas cinco variables con tus valores y guarda (`Ctrl+O`, Enter, `Ctrl+X`):

```yaml
vault_become_password: "CONTRASEÑA_SUDO_DEL_USUARIO"
vault_mariadb_root_password: "CLAVE_ROOT_MARIADB"
vault_samba_password: "CLAVE_USUARIO_SAMBA"
vault_postgres_password: "CLAVE_POSTGRES"
vault_pg_app_password: "CLAVE_USUARIO_APPUSER"
```

**2. Guarda la contraseña del vault en un archivo protegido** (así Ansible no la pide cada vez):

```bash
echo "CONTRASEÑA_DEL_VAULT" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

El archivo `ansible.cfg` ya apunta a `~/.vault_pass`.

**3. Comprueba que el vault se lee bien:**

```bash
ansible-vault view inventory/group_vars/all/vault.yml
```

> Nunca subas `~/.vault_pass` ni llaves privadas a GitHub. El `.gitignore` ya excluye `.vault_pass`.

---

## 5. Ejecución

Desde la carpeta del proyecto en el nodo de control:

```bash
# 1. Probar la conexión (debe responder "pong")
ansible all -m ping

# 2. Validar la sintaxis
ansible-playbook site.yml --syntax-check

# 3. Simulación (opcional, no cambia nada)
ansible-playbook site.yml --check

# 4. Despliegue real
ansible-playbook site.yml
```

La primera ejecución tarda varios minutos porque instala paquetes. Si la repites, todo debe salir en `ok` con `changed=0` (el despliegue es idempotente).

Para ejecutar solo un role, crea un playbook temporal que lo incluya, o comenta los demás en `site.yml`.

---

## 6. Verificación de cada servicio

Sustituye `IP` por la del servidor destino.

**Apache y PHP**
```bash
curl -I http://IP/                  # 200 OK
```

**phpMyAdmin**: abre `http://IP/phpMyAdmin/` en el navegador. Usuario `root` y la contraseña `vault_mariadb_root_password`.

**MariaDB** (en el servidor): `mariadb -u root -p`

**Samba**
```bash
smbclient -L //IP -U sambauser                     # lista el recurso "compartido"
smbclient //IP/compartido -U sambauser -c "put /etc/hostname prueba.txt; ls"
```
En el servidor, `ls -l /srv/samba/compartido` debe mostrar el archivo con permisos `-rw-rw-r--` (`create mask = 0664`).

**FTP**
```bash
ftp IP
```
Entra con un usuario local del servidor (por ejemplo, el de `ansible_user`). Dentro del cliente escribe `passive` (debe decir `Passive mode on`) y luego `ls`.

**PostgreSQL** (conexión remota cifrada)
```bash
psql "host=IP dbname=appdb user=appuser sslmode=require"
```
Debe mostrar `SSL connection (protocol: TLSv1.3 ...)`. Si usas `sslmode=disable`, el servidor debe rechazar la conexión.

**Librería PHP de PostgreSQL**
```bash
ssh USUARIO@IP "rpm -q php8-pgsql"
```

---

## 7. Solución de problemas

| Síntoma | Causa probable y solución |
|---|---|
| `Attempting to decrypt but no vault secrets found` | Falta `~/.vault_pass` o la línea `vault_password_file` en `ansible.cfg`. |
| `Decryption failed` | La contraseña de `~/.vault_pass` no coincide con la del vault. |
| `UNREACHABLE` / `Permission denied` | La llave SSH no está copiada, o `ansible_user` / `ansible_host` son incorrectos. |
| Falla `sudo` (`Missing sudo password`) | `vault_become_password` no es la contraseña de sudo del usuario. |
| Apache muestra `403 Forbidden` | Falta `index.html`; el role `lamp` lo crea. Si persiste, revisa SELinux (`getenforce`). |
| `ls` en FTP da `425 Failed to establish connection` | El cliente está en modo activo: escribe `passive` antes de `ls`. |
| Un paquete no existe | Los nombres cambian entre versiones. Busca con `zypper search NOMBRE` en el servidor y ajústalo en `tasks/main.yml` (por ejemplo `python313-PyMySQL` o `python313-psycopg2` dependen de la versión de Python). |
| `couldn't resolve module/action 'community.xxx'` | Faltan colecciones: `ansible-galaxy collection install -r requirements.yml`. |

---

## 8. Estructura del proyecto

```
.
├── ansible.cfg                  # Optimización (pipelining, forks, caché de facts) y vault
├── requirements.yml             # Colecciones de Ansible necesarias
├── site.yml                     # Playbook principal con los roles
├── inventory/
│   ├── hosts.yml                # Solo la IP del servidor
│   └── group_vars/all/
│       ├── vars.yml             # Variables (referencian al vault)
│       └── vault.yml            # Contraseñas cifradas (Ansible Vault)
└── roles/
    ├── mariadb/
    ├── lamp/
    ├── samba/                   # templates/smb.conf.j2
    ├── ftp/                     # templates/vsftpd.conf.j2
    └── postgresql/
```

Las plantillas de configuración (`smb.conf.j2`, `vsftpd.conf.j2`) están versionadas en el repositorio.

---

## 9. Notas de seguridad

- `host_key_checking = False` en `ansible.cfg` facilita el laboratorio, pero en producción conviene activarlo.
- Usa contraseñas largas, sobre todo para el vault.
- `pg_remote_network: 0.0.0.0/0` permite conectarse desde cualquier IP (siempre con SSL). Limítalo a tu red.
- El certificado SSL de PostgreSQL es autofirmado: sirve para cifrar, no para verificar la identidad del servidor.
- El FTP configurado no usa TLS: usuario, contraseña y archivos viajan sin cifrar. Úsalo solo en redes de laboratorio.
