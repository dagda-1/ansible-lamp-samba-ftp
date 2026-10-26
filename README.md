# Despliegue LAMP + Samba + PostgreSQL con Ansible (openSUSE 16.0)

Proyecto de Ansible que configura de forma remota una máquina **destino** desde una máquina **Master**.

Instala y configura:

- **Apache + PHP + phpMyAdmin** (rol `lamp`)
- **MariaDB** con contraseña de superusuario (rol `mariadb`)
- **Samba** con usuario de sistema, usuario Samba, directorio compartido y `create mask` (rol `samba`)
- **PostgreSQL** con librería de PHP, contraseña, usuario nuevo, conexión remota y cifrado SSL (rol `postgresql`)

Las contraseñas están cifradas con **Ansible Vault**, y el archivo de hosts no tiene usuario ni contraseña visibles.

---

## 1. Estructura del proyecto

```
.
├── ansible.cfg
├── site.yml
├── inventory/
│   ├── hosts.yml                 <- aquí va la IP del destino
│   └── group_vars/
│       └── all/
│           ├── vars.yml          <- usuario y variables
│           └── vault.yml         <- contraseñas cifradas
└── roles/
    ├── mariadb/
    ├── lamp/
    ├── samba/                    <- incluye templates/smb.conf.j2
    └── postgresql/
```

---

## 2. Qué hacer en la máquina DESTINO

El destino es la máquina que recibe la instalación (openSUSE 16.0). Se configura **una sola vez**, desde su consola.

**2.1. Poner contraseña a root** (anótala, la necesitarás en el Vault):
```bash
sudo passwd root
```

**2.2. Permitir la entrada de root por SSH y activar SSH:**
```bash
echo "PermitRootLogin yes" | sudo tee /etc/ssh/sshd_config.d/99-root.conf
sudo systemctl enable --now sshd
sudo systemctl restart sshd
```

**2.3. Ver la IP de la máquina** (la usarás en el Master):
```bash
ip a
```

Con esto el destino queda listo. No hay que instalar nada más: Ansible lo hace todo.

---

## 3. Qué hacer en la máquina MASTER

El Master es la máquina desde donde se ejecuta Ansible.

**3.1. Instalar lo necesario:**
```bash
sudo zypper install ansible sshpass git
ansible-galaxy collection install community.mysql community.general ansible.posix community.postgresql
```

**3.2. Descargar el proyecto y entrar a la carpeta:**
```bash
git clone URL-DEL-REPOSITORIO
cd NOMBRE-DE-LA-CARPETA
```

Todos los comandos siguientes se ejecutan **desde esta carpeta** (la que contiene `ansible.cfg` y `site.yml`).

---

## 4. Cambiar la IP del destino

Edita el archivo de hosts:
```bash
nano inventory/hosts.yml
```

Deja el contenido así, poniendo **la IP de tu máquina destino**:
```yaml
all:
  children:
    servers:
      hosts:
        opensuse01:
          ansible_host: 192.168.0.201
```

Guarda con `Ctrl+O`, Enter, y sal con `Ctrl+X`. El grupo debe llamarse `servers`, porque `site.yml` usa ese nombre.

---

## 5. Contraseñas y Ansible Vault

Las contraseñas están en `inventory/group_vars/all/vault.yml`, cifrado con Ansible Vault.

### Clave del Vault

```
mundolibre
```

Es la clave que pide Ansible cada vez que se abre el Vault o se ejecuta el playbook.

### Ver el contenido del Vault
```bash
ansible-vault view inventory/group_vars/all/vault.yml --ask-vault-pass
```

### Editar el Vault
```bash
export EDITOR=nano
ansible-vault edit inventory/group_vars/all/vault.yml --ask-vault-pass
```

### Variables que contiene

| Variable | Para qué sirve |
|---|---|
| `vault_become_password` | Contraseña de **root del destino**, que Ansible usa para entrar por SSH |
| `vault_mariadb_root_password` | Contraseña de root de MariaDB |
| `vault_samba_password` | Contraseña del usuario de Samba |
| `vault_postgres_password` | Contraseña del usuario `postgres` |
| `vault_pg_app_password` | Contraseña del usuario nuevo `appuser` de PostgreSQL |

### IMPORTANTE: si la contraseña de root de tu destino es distinta

Si la contraseña de root de **tu** máquina destino no es la misma que está guardada, hay que cambiarla en el Vault. Es la variable `vault_become_password`:

```bash
export EDITOR=nano
ansible-vault edit inventory/group_vars/all/vault.yml --ask-vault-pass
```

Escribe la clave `mundolibre`, y en el archivo cambia solo esta línea por la contraseña de root de tu destino:
```yaml
vault_become_password: "TU-CONTRASEÑA-DE-ROOT"
```

Guarda con `Ctrl+O`, Enter, y sal con `Ctrl+X`. Las demás líneas déjalas como están.

Si no se cambia, aparece el error `Permission denied` al conectar.

---

## 6. Variables visibles (`vars.yml`)

Archivo `inventory/group_vars/all/vars.yml`:

```yaml
ansible_user: root
ansible_become: true
ansible_password: "{{ vault_become_password }}"
mariadb_root_password: "{{ vault_mariadb_root_password }}"
samba_user: sambauser
samba_password: "{{ vault_samba_password }}"
samba_share_path: /srv/samba/compartido

postgres_password: "{{ vault_postgres_password }}"
pg_app_user: appuser
pg_app_password: "{{ vault_pg_app_password }}"
pg_app_db: appdb
```

- `ansible_user`: usuario con el que Ansible entra por SSH al destino.
- `ansible_become: true`: necesario para que el rol de PostgreSQL pueda ejecutar tareas como el usuario `postgres`.
- En este archivo no hay contraseñas en texto plano: solo referencias al Vault.

---

## 7. Ejecutar

**7.1. Probar la conexión:**
```bash
ansible all -m ping --ask-vault-pass
```
Debe responder `pong` en verde.

**7.2. Revisar la sintaxis:**
```bash
ansible-playbook site.yml --syntax-check --ask-vault-pass
```

**7.3. Ejecutar la instalación completa:**
```bash
ansible-playbook site.yml --ask-vault-pass
```
Puede tardar varios minutos. Al final, en el resumen (`PLAY RECAP`) debe aparecer `failed=0` y `unreachable=0`.

**7.4. Ejecutar solo el rol de PostgreSQL:**
```bash
ansible servers -m ansible.builtin.include_role -a name=postgresql --ask-vault-pass
```

**Opcional: no escribir la clave cada vez.** Crea un archivo con la clave (no debe subirse a GitHub):
```bash
echo "mundolibre" > ~/.vault_pass
chmod 600 ~/.vault_pass
```
Con ese archivo, los comandos funcionan sin `--ask-vault-pass`.

---

## 8. Cómo comprobar que funcionó

**Apache y PHP:** abre en el navegador `http://IP-DEL-DESTINO/`

**phpMyAdmin:** abre `http://IP-DEL-DESTINO/phpMyAdmin`

**MariaDB** (en el destino):
```bash
mysql -u root -p
```

**Samba** (desde el Master):
```bash
smbclient -L //IP-DEL-DESTINO -U sambauser
```
Desde Windows: `\\IP-DEL-DESTINO\compartido`

**Librería de PHP para PostgreSQL** (desde el Master):
```bash
ansible servers -m ansible.builtin.shell -a "php -m | grep pgsql" --ask-vault-pass
```

**PostgreSQL con conexión remota y cifrado** (desde el Master):
```bash
sudo zypper install postgresql18
psql "host=IP-DEL-DESTINO user=appuser dbname=appdb sslmode=require"
```
La contraseña de `appuser` es `vault_pg_app_password` (se ve con el comando de `ansible-vault view`). Si aparece `appdb=>`, funciona. Dentro, `\conninfo` muestra que la conexión usa SSL, y `\q` sale.

---

## 9. Problemas frecuentes

| Error | Causa y solución |
|---|---|
| `UNREACHABLE ... Permission denied` | La contraseña de root del Vault no coincide con la del destino, o falta `PermitRootLogin yes`. Revisa los apartados 2 y 5 |
| `Connection refused` / `timed out` | SSH no está activo en el destino, o la IP es incorrecta. Ejecuta allá `sudo systemctl enable --now sshd` y revisa la IP con `ip a` |
| `site.yml could not be found` | No estás en la carpeta del proyecto. Entra con `cd` y comprueba con `ls` |
| `Decryption failed` | La clave del Vault es incorrecta |
| `variable is undefined` | Falta una variable en el Vault (revisa la tabla del apartado 5) |
| Paquete no encontrado | El nombre cambia en openSUSE. Búscalo con `zypper search NOMBRE` en el destino y corrígelo en el rol |
| `la autentificación Peer falló para el usuario «postgres»` | `ansible_become` debe estar en `true` en `vars.yml` |
| `Host key verification failed` | Ejecuta una vez `ssh root@IP-DEL-DESTINO`, responde `yes` y vuelve a probar |

---

## 10. Resumen rápido

1. **Destino:** contraseña de root, `PermitRootLogin yes`, SSH activo, anotar la IP.
2. **Master:** instalar `ansible`, `sshpass`, `git` y las colecciones.
3. Cambiar la IP en `inventory/hosts.yml`.
4. Si la contraseña de root es otra, cambiar `vault_become_password` en el Vault (clave: `mundolibre`).
5. `ansible all -m ping --ask-vault-pass`
6. `ansible-playbook site.yml --ask-vault-pass`
