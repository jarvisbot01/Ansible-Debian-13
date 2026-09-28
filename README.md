# Ansible Debian 13 Automation

Este repositorio contiene la automatización con Ansible para aprovisionar y configurar un servidor con **Debian 13 (Trixie)**, instalando y configurando **Nginx**, **Docker**, **Docker Compose Plugin**, **Docker Buildx Plugin** y desplegando un archivo web de prueba.

---

## 📁 Estructura del Proyecto

```text
.
├── ansible.cfg          # Configuración global de Ansible
├── inventory.yml        # Inventario de hosts y grupos
├── playbooks/
│   ├── files/           # Archivos estáticos a transferir (ej. index.html)
│   └── site.yml         # Playbook principal de aprovisionamiento
├── vault.yml.example    # Plantilla de ejemplo para variables encriptadas
└── vault.yml            # Archivo encriptado con Ansible Vault (credenciales/IPs)
```

---

## 🚀 Requisitos Previos

1. **Ansible instalado** en tu máquina de control (por ejemplo, en **Arch Linux** usando `pacman`; para otras distros consulta la [guía oficial de instalación](https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html)):
   ```bash
   # En Arch Linux:
   sudo pacman -Syu ansible ansible-lint

   # En Debian/Ubuntu:
   sudo apt update && sudo apt install -y ansible ansible-lint
   ```
2. **Acceso SSH y ssh-agent (por sesión de terminal)**:
   Si tu llave privada tiene contraseña (*passphrase*), debe estar cargada en el agente SSH (`ssh-agent`) antes de ejecutar Ansible para evitar errores de autenticación (`Permission denied`).
   
   > **Ten en cuenta:** `ssh-agent` funciona **por sesión de terminal** (cada vez que abres una nueva ventana, pestaña o sesión de terminal, deberás cargarla):
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```
   > 💡 **Alternativa recomendada (100% automático):**  
   > 1. Asegura que el agente SSH inicie automáticamente en cada terminal agregando esto a tu `~/.bashrc` o `~/.zshrc`:
   >    ```bash
   >    if [ -z "$SSH_AUTH_SOCK" ]; then
   >        eval "$(ssh-agent -s)" > /dev/null
   >    fi
   >    ```
   > 2. Configura tu cliente SSH en `~/.ssh/config` para cargar la llave la primera vez que se use:
   >    ```ssh-config
   >    Host *
   >        AddKeysToAgent yes
   >        IdentityFile ~/.ssh/id_ed25519
   >    ```
   > Con esta combinación, al abrir cualquier terminal el agente ya estará activo, y la primera vez que Ansible o SSH hagan una conexión te pedirán la contraseña (*passphrase*) una única vez para esa sesión.
3. Servidor destino con **Debian 13**.

---

## 🔐 Configuración de Ansible Vault

El inventario utiliza variables definidas en `vault.yml` (`vault_ansible_host`, `vault_ansible_user`, `vault_ssh_private_key_path`).

1. Copia la plantilla de ejemplo:
   ```bash
   cp vault.yml.example vault.yml
   ```
2. Edita `vault.yml` con tus datos reales y encripta el archivo:
   ```bash
   ansible-vault encrypt vault.yml
   ```
   *(O si ya tienes el archivo encriptado, puedes editarlo con `ansible-vault edit vault.yml`)*

---

## 📡 Probar Conectividad

Para validar la conexión SSH y la autenticación contra los servidores definidos en el grupo `webservers`, ejecuta el siguiente comando:

```bash
ansible webservers -m ping -e @vault.yml --ask-vault-pass
```

> [!TIP]
> **Importante:** Recuerda tener tu llave SSH cargada en el agente (`ssh-add ~/.ssh/id_ed25519`) antes de correr este comando o el playbook, de lo contrario la autenticación fallará si la llave requiere contraseña.

> **Nota:** Si utilizas un archivo con la contraseña de Vault configurado o guardado (por ejemplo `--vault-password-file .vault_pass`), puedes prescindir del parámetro `--ask-vault-pass`.

---

## 🛠️ Ejecución del Playbook

Para aplicar toda la configuración en el servidor remoto:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### ¿Qué realiza el Playbook?
- Actualiza la caché de paquetes `apt`.
- Instala paquetes esenciales (`curl`, `ca-certificates`).
- Instala y habilita **Nginx**.
- Instala **Docker** mediante su script oficial y añade los plugins `docker-compose-plugin` y `docker-buildx-plugin`.
- Añade el usuario de conexión al grupo `docker`.
- Valida el estado activo de los servicios en `systemd`.
- Despliega un archivo `index.html` en `/var/www/html/index.html`.

---

## 🌐 Validar Conexión Web (HTTP)

Una vez finalizada la ejecución del playbook, puedes verificar que el servidor Nginx está respondiendo correctamente realizando peticiones con `curl`:

1. **Obtener solo las cabeceras HTTP (`-I`):**
   ```bash
   curl -I http://<IP_DEL_SERVIDOR>
   ```
   *(Permite verificar el código de respuesta HTTP `200 OK` y el encabezado `Server: nginx`)*

2. **Obtener el contenido en modo silencioso (`-s`):**
   ```bash
   curl -s http://<IP_DEL_SERVIDOR>
   ```
   *(Muestra directamente el contenido del archivo `index.html` sin barras de progreso)*

---

## 🔐 Ansible Vault Password File

En lugar de usar `--ask-vault-pass` en cada ejecución, puedes guardar la contraseña de Vault en un archivo y usar `--vault-password-file`.

### Crear archivo de contraseña

```bash
echo "TU_CONTRASEÑA_DE_VAULT" > .vault_pass
chmod 600 .vault_pass
```

> **Reemplaza:**
> - `TU_CONTRASEÑA_DE_VAULT` con la contraseña real de tu archivo `vault.yml`
> - `chmod 600 .vault_pass` otorga permisos de solo lectura para el usuario actual, lo cual es más seguro

### Configurar en `ansible.cfg` (Automático)

Como ya se encuentra configurado en `ansible.cfg`:

```ini
[defaults]
vault_password_file = .vault_pass
```

Una vez que exista `.vault_pass`, ya no necesitas pasar ninguna bandera de Vault; puedes correr directamente:

```bash
ansible-playbook playbooks/site.yml
```

O si prefieres indicarlo explícitamente por línea de comandos (aunque con la configuración anterior no es necesario):

```bash
ansible-playbook playbooks/site.yml --vault-password-file .vault_pass
```

### Eliminar archivo de contraseña

Cuando ya no lo necesites:

```bash
rm .vault_pass
```

> [!WARNING]
> **Nunca** subas el archivo `.vault_pass` a sistemas de control de versiones como Git. Asegúrate de que esté listado en tu archivo `.gitignore`.
