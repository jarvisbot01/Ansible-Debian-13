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
├── vault.yml.example    # Plantilla de ejemplo para variables cifradas
└── vault.yml            # Archivo cifrado con Ansible Vault (credenciales/IPs)
```

---

## 🚀 Requisitos Previos

1. **Ansible instalado** en tu máquina de control (por ejemplo, en **Arch Linux** usando `pacman`; para otras distribuciones consulta la [guía oficial de instalación](https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html)):
   ```bash
   # En Arch Linux:
   sudo pacman -Syu ansible ansible-lint

   # En Debian/Ubuntu:
   sudo apt update && sudo apt install -y ansible ansible-lint
   ```
2. **Acceso SSH y ssh-agent (por sesión de terminal)**:
   Si tu llave privada tiene contraseña (*passphrase*), debe estar cargada en el agente SSH (`ssh-agent`) antes de ejecutar Ansible para evitar errores de autenticación (`Permission denied`).
   
   > ℹ️ **Nota:** `ssh-agent` funciona **por sesión de terminal**. Cada vez que abras una nueva ventana, pestaña o sesión de terminal, deberás cargarla:
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```
   > 💡 **Alternativa recomendada (100% automático):**  
   > 1. Inicia el agente SSH automáticamente en cada terminal agregando esto a tu `~/.bashrc` o `~/.zshrc`:
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
   > Con esta combinación, al abrir cualquier terminal el agente ya estará activo, y la primera vez que Ansible o SSH realicen una conexión te pedirán la contraseña (*passphrase*) una única vez para esa sesión.
3. Servidor destino con **Debian 13**.

---

## 🔐 Configuración de Ansible Vault

El inventario utiliza variables definidas en `vault.yml` (`vault_ansible_host`, `vault_ansible_user`, `vault_ssh_private_key_path`).

1. Copia la plantilla de ejemplo:
   ```bash
   cp vault.yml.example vault.yml
   ```
2. Edita `vault.yml` con tus datos reales y cifra el archivo:
   ```bash
   ansible-vault encrypt vault.yml
   ```
   *(O si ya tienes el archivo cifrado, puedes editarlo con `ansible-vault edit vault.yml`)*.

---

## 🔑 Archivo de Contraseña de Vault (`.vault_pass`)

Para evitar ingresar la contraseña de Vault interactivamente en cada comando, puedes guardarla en un archivo local `.vault_pass`.

### Crear archivo de contraseña

```bash
echo "TU_CONTRASEÑA_DE_VAULT" > .vault_pass
chmod 600 .vault_pass
```

- Reemplaza `TU_CONTRASEÑA_DE_VAULT` con la contraseña asignada a tu archivo `vault.yml`.
- El comando `chmod 600 .vault_pass` restringe los permisos a solo lectura y escritura para tu usuario, mejorando la seguridad.

> ℹ️ **Nota:** En [ansible.cfg](ansible.cfg) ya está configurada la directiva:
> ```ini
> [defaults]
> vault_password_file = .vault_pass
> ```
> Por lo tanto, mientras exista `.vault_pass`, Ansible descifrará automáticamente los archivos de Vault sin requerir flags adicionales.

> ⚠️ **Advertencia:** **Nunca** subas el archivo `.vault_pass` a repositorios o sistemas de control de versiones. Asegúrate de que permanezca listado en tu [.gitignore](.gitignore).

---

## 📡 Probar Conectividad

Para validar la conexión SSH y la autenticación contra los servidores del grupo `webservers`:

```bash
# Con .vault_pass configurado:
ansible webservers -m ping -e @vault.yml

# O solicitando la contraseña manualmente:
ansible webservers -m ping -e @vault.yml --ask-vault-pass
```

> ℹ️ **Nota:** En comandos ad-hoc como `ping`, se requiere pasar explícitamente `-e @vault.yml` para que Ansible cargue las variables de conexión del host cifradas en el Vault.

> 💡 **Consejo:** Recuerda tener tu llave SSH cargada en el agente (`ssh-add ~/.ssh/id_ed25519`) antes de ejecutar este comando o el playbook; de lo contrario, la autenticación fallará si la llave está protegida con contraseña.

---

## 🔍 Validación de Sintaxis y Linting

Antes de ejecutar las tareas en el servidor remoto, puedes verificar la calidad y sintaxis de los playbooks:

```bash
# Verificar buenas prácticas y formato con ansible-lint:
ansible-lint

# Comprobar la sintaxis del playbook:
ansible-playbook playbooks/site.yml --syntax-check
```

---

## 🛠️ Ejecución del Playbook

Para aplicar toda la configuración en el servidor remoto:

```bash
# Con .vault_pass configurado:
ansible-playbook playbooks/site.yml

# O solicitando la contraseña manualmente:
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### ¿Qué realiza el Playbook?
- Actualiza la caché y los paquetes de `apt`.
- Instala paquetes esenciales (`curl`, `ca-certificates`).
- Instala y habilita **Nginx**.
- Instala **Docker** mediante su script oficial y añade los plugins `docker-compose-plugin` y `docker-buildx-plugin`.
- Añade el usuario de conexión al grupo `docker`.
- Valida el estado activo de los servicios en `systemd`.
- Despliega un archivo `index.html` en `/var/www/html/index.html`.

---

## 🌐 Validar Conexión Web (HTTP)

Una vez finalizada la ejecución del playbook, puedes verificar que el servidor Nginx responde correctamente realizando peticiones con `curl`:

1. **Obtener solo las cabeceras HTTP (`-I`):**
   ```bash
   curl -I http://<IP_DEL_SERVIDOR>
   ```
   *(Permite verificar el código de respuesta HTTP `200 OK` y el encabezado `Server: nginx`)*.

2. **Obtener el contenido en modo silencioso (`-s`):**
   ```bash
   curl -s http://<IP_DEL_SERVIDOR>
   ```
   *(Muestra directamente el contenido del archivo `index.html` sin barras de progreso)*.
