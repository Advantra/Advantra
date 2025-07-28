## Documentación Detallada: Infraestructura y Operaciones DevOps

Esta guía está pensada para que el equipo de Advantra pueda provisionar, configurar y desplegar entornos _dev_, _pre_ y _prod_ para cualquier cliente, tomando como ejemplo la aplicación **NextDocument**.

---

### 1. Creación de cuentas y provisión inicial

1. **Cuenta de Gmail del cliente**: Crear una cuenta Gmail genérica a nombre del cliente (e.g., `nextdocument@gmail.com`).
2. **Google Cloud (GCP)**:

   - Con la cuenta Gmail recién creada, dar de alta un proyecto en GCP.
   - Registrar método de pago (temporal: usa nuestra tarjeta y teléfono); marcar recordatorio para cambiar a los datos del cliente antes de entrega.

3. **Supabase / MongoDB Atlas**:

   - Crear proyecto en Supabase si la base de datos es relacional, o en MongoDB Atlas si es NoSQL.
   - Asociar el proyecto a la cuenta Gmail del cliente.

---

### 2. Gestión de identidades y accesos (IAM)

- **Roles y permisos mínimos**:

  - Crear una **Service Account** en GCP para automaciones de CI/CD; asignar roles: _Compute Admin_, _Storage Admin_, _DNS Admin_.
  - Los desarrolladores: asignarles rol _Compute Instance Admin_ sólo en recursos del proyecto.

- **Claves y secretos**:

  - Generar y descargar la clave JSON de la Service Account;
  - Guardar en **GitHub Secrets**: `GCP_SA_KEY`, `GCP_PROJECT_ID`.

---

### 3. Red y seguridad de red

1. **VPC y subredes**:

   - Crear una VPC dedicada al proyecto con subredes públicas y privadas si se desea aislar bases de datos.

2. **Firewall (Cloud Firewall)**:

   - Regla SSH: puerto 22 abierto sólo a IPs autorizadas (lista en documento compartido).
   - HTTP (80) y HTTPS (443): abiertos a todo el mundo sólo en _pre_ y _prod_.
   - Denegar todo lo demás por defecto.

3. **Bastion Host (opcional)**:

   - VM adicional en subred pública con única IP externa.
   - Sólo puerto 22 abierto desde la lista de IPs autorizadas.
   - Usar para SSH-chaining a las VMs de la subred privada.

---

### 4. Provisionamiento de instancias VM

| Entorno  | Tipo de VM      | vCPU / RAM | Disco SSD | IP Estática | Etiquetas               |
| -------- | --------------- | ---------- | --------- | ----------- | ----------------------- |
| **dev**  | `e2-medium`     | 2/4 GB     | 50 GB     | Sí          | `dev-app-nextdocument`  |
| **pre**  | `e2-standard-2` | 2/8 GB     | 100 GB    | Sí          | `pre-app-nextdocument`  |
| **prod** | `e2-standard-4` | 4/16 GB    | 200 GB    | Sí          | `prod-app-nextdocument` |

---

### 5. Instalación de stack base en cada VM

Ejecutar en **cada** VM:

```bash
# Actualizar SO y paquetes
sudo apt update && sudo apt upgrade -y

# Instalar software requerido
sudo apt install -y git nginx nodejs npm docker.io docker-compose \
  certbot python3-certbot-nginx ufw

# Instalar PM2 (gestión de procesos Node.js)
sudo npm install -g pm2
```

---

### 6. Configuración de firewall (UFW) en VMs

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH sólo desde IPs autorizadas
echo "sshd: X.X.X.X, Y.Y.Y.Y" | sudo tee /etc/ufw/applications.d/ssh-custom
sudo ufw allow from X.X.X.X to any port 22
# Abrir HTTP y HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
```

---

### 7. Configuración de dominios y DNS

1. **Registrar dominios** (e.g., `nextdocument.com`) y gestionar DNS en proveedor (IONOS, GoDaddy, Cloudflare).
2. **Crear registros A** para cada subdominio:

   - `dev-app.nextdocument.com` → IP VM dev
   - `pre-app.nextdocument.com` → IP VM pre
   - `app.nextdocument.com` → IP VM prod

---

### 8. Despliegue de la aplicación

1. **Clonar repositorio**:

   ```bash
   git clone git@github.com:cliente/nextdocument.git /opt/nextdocument
   cd /opt/nextdocument
   ```

2. **Variables de entorno**:

   - Copiar plantilla: `cp .env.example .env.dev` (o `.env.pre`, `.env.prod`).
   - Editar con `nano .env.dev` y añadir configuraciones específicas.

3. **Instalar dependencias**:

   ```bash
   npm install
   ```

4. **Iniciar con PM2**:

   ```bash
   pm2 start index.js --name nextdocument-dev -- env .env.dev
   pm2 save && pm2 startup
   ```

---

### 9. Configuración de NGINX y SSL

1. **Archivo de configuración**:

   ```bash
   sudo tee /etc/nginx/sites-available/nextdocument-dev << 'EOF'
   server {
     listen 80;
     server_name dev-app.nextdocument.com;
     return 301 https://$host$request_uri;
   }
   server {
     listen 443 ssl http2;
     server_name dev-app.nextdocument.com;

     client_max_body_size 50M;

     ssl_certificate /etc/letsencrypt/live/dev-app.nextdocument.com/fullchain.pem;
     ssl_certificate_key /etc/letsencrypt/live/dev-app.nextdocument.com/privkey.pem;
     include /etc/letsencrypt/options-ssl-nginx.conf;
     ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

     location / {
       proxy_pass http://localhost:3000;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection 'upgrade';
       proxy_set_header Host $host;
     }
   }
   EOF
   ```

2. **Habilitar y probar**:

   ```bash
   sudo ln -s /etc/nginx/sites-available/nextdocument-dev /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```

3. **Obtener certificado**:

   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   sudo certbot --nginx -d dev-app.nextdocument.com --non-interactive --agree-tos -m admin@nextdocument.com
   ```

4. **Renovación automática**: verificar timer systemd y ejecutar `sudo certbot renew --dry-run`.

---

### 10. CI/CD con GitHub Actions

En el repo crear `.github/workflows/deploy.yml`:

```yaml
name: CI/CD
on:
  push:
    branches:
      - dev
      - main
  workflow_dispatch:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Instalar dependencias
      run: npm ci
    - name: Ejecutar tests
      run: npm test

  deploy-dev:
    if: github.ref == 'refs/heads/dev'
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - name: Desplegar en dev
        run: |
          gcloud compute ssh dev-app-nextdocument --zone=us-central1-a --command='cd /opt/nextdocument && git pull && npm install && pm2 restart nextdocument-dev'

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - name: Desplegar en prod
        run: |
          gcloud compute ssh prod-app-nextdocument --zone=us-central1-a --command='cd /opt/nextdocument && git pull && npm install && pm2 restart nextdocument-prod'
```

---

### 11. Backups y monitoreo

1. **Backups de BD**:

   - Script diario en cron (`02:00`):

     ```bash
     #!/bin/bash
     TIMESTAMP=$(date +"%F_%H-%M")
     pg_dump --dbname=${DB_URL} > /backups/nextdoc_$TIMESTAMP.sql
     mongodump --uri="${MONGO_URI}" --archive=/backups/nextdoc_mongo_$TIMESTAMP.archive
     aws s3 cp /backups/ s3://bucket-backups-nextdocument/ --recursive
     ```

2. **Logs y métricas**:

   - Instalar **Filebeat** que envíe logs a ELK o **Prometheus Node Exporter**.
   - Dashboards básicos en Grafana para uso de CPU, memoria y tráfico HTTP.

---

### 12. Transferencia al cliente

1. **Datos de acceso**:

   - Gmail del proyecto y contraseña.
   - Credenciales GCP (transferir billing).
   - Acceso a Supabase/Atlas.
   - Repositorio GitHub: invitar al cliente y transferir organización o repo.

2. **Documentación**:

   - Este playbook en PDF/Markdown.
   - Guía de recuperación ante desastres.
   - Contacto de soporte Advantra.

---

### Anexo: Estructura de ficheros de configuración

```text
/opt/nextdocument/
├── .env.dev
├── .env.pre
├── .env.prod
├── index.js
├── ecosystem.config.js    # PM2
└── .github/workflows/
    └── deploy.yml
```

---

_Fin de la guía._

---

## 13. Publicación de la guía en GitHub

Para que el equipo pueda acceder fácilmente a esta documentación desde la cuenta de GitHub de Advantra, sigue estos pasos:

1. **Crear o seleccionar un repositorio**:

   - En GitHub, bajo la organización o cuenta de Advantra, crea un nuevo repositorio llamado por ejemplo `infraestructura-devops`.
   - Marca la opción de **Initialize this repository with a README** para crear un primer commit.

2. **Añadir la guía en formato Markdown**:

   - Clona el repositorio localmente:

     ```bash
     git clone git@github.com:Advantra/infraestructura-devops.git
     cd infraestructura-devops
     ```

   - Copia el contenido de esta guía en un archivo `README.md` o en una carpeta `docs/infraestructura.md`:

     ```bash
     # Como README.md
     nano README.md
     # Pegar contenido de la guía y guardar

     # O dentro de docs/
     mkdir -p docs
     nano docs/infraestructura.md
     # Pegar contenido de la guía
     ```

3. **Commits y push**:

   ```bash
   git add README.md       # o docs/infraestructura.md
   git commit -m "Añadir guía detallada de Infraestructura y DevOps"
   git push origin main
   ```

4. **Configuración de GitHub Pages (opcional)**:

   - En la pestaña **Settings > Pages**, elige la rama `main` y la carpeta raíz o `docs/` como fuente.
   - Guarda y espera unos minutos; la documentación quedará disponible en `https://advantra.github.io/infraestructura-devops/`.

5. **Control de versiones y actualizaciones**:

   - Para futuras modificaciones, edita directamente el archivo en GitHub o mediante pull requests.
   - Usa etiquetas de Git (`git tag v1.0`) para marcar versiones estables de la guía.

Con estos pasos, tu equipo y cualquier colaborador tendrán acceso inmediato a la guía desde el repositorio de Advantra en GitHub.
