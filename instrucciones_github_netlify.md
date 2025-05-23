# Instrucciones para GitHub y Netlify

## 1. Descargar el proyecto

Ya hemos creado un archivo ZIP con todo el proyecto. Puedes descargarlo desde:
https://work-1-vzupolrppelmpwbo.prod-runtime.all-hands.dev/selvadentro_ai_project.zip

## 2. Subir a GitHub

1. Crea un nuevo repositorio en GitHub:
   - Ve a https://github.com/new
   - Nombre del repositorio: `selvadentro-ai-project`
   - Descripción: `Proyecto de implementación de agentes de IA para ventas de Selvadentro Tulum`
   - Selecciona "Public" o "Private" según prefieras
   - Haz clic en "Create repository"

2. Sube el proyecto a GitHub:
   ```bash
   # Descomprime el archivo ZIP
   unzip selvadentro_ai_project.zip
   
   # Entra al directorio
   cd selvadentro_ai_project
   
   # Inicializa Git
   git init
   git branch -m main
   
   # Añade todos los archivos
   git add .
   
   # Haz el primer commit
   git commit -m "Initial commit: Selvadentro AI Project"
   
   # Conecta con tu repositorio de GitHub (reemplaza USERNAME con tu nombre de usuario)
   git remote add origin https://github.com/USERNAME/selvadentro-ai-project.git
   
   # Sube el proyecto
   git push -u origin main
   ```

## 3. Vincular con Netlify

1. Crea una cuenta en Netlify si aún no tienes una:
   - Ve a https://app.netlify.com/signup

2. Conecta tu repositorio de GitHub:
   - Inicia sesión en Netlify
   - Haz clic en "New site from Git"
   - Selecciona "GitHub" como proveedor de Git
   - Autoriza a Netlify para acceder a tus repositorios
   - Busca y selecciona tu repositorio `selvadentro-ai-project`

3. Configura las opciones de despliegue:
   - Branch to deploy: `main`
   - Base directory: (déjalo en blanco)
   - Build command: (déjalo en blanco)
   - Publish directory: (déjalo en blanco)

4. Haz clic en "Deploy site"

5. Espera a que se complete el despliegue y obtendrás una URL de Netlify (por ejemplo, `https://selvadentro-ai.netlify.app`)

6. Personaliza el dominio (opcional):
   - Ve a "Domain settings"
   - Haz clic en "Custom domains"
   - Puedes usar un subdominio de Netlify o configurar tu propio dominio personalizado

## Notas importantes

- El sitio desplegado en Netlify mostrará correctamente el HTML y las imágenes, pero los archivos Markdown (.md) se mostrarán como texto plano.
- Para una mejor visualización de los archivos Markdown, considera usar una herramienta como Docsify o Jekyll para convertir los archivos Markdown en HTML.
- Puedes actualizar el sitio simplemente haciendo push a tu repositorio de GitHub. Netlify detectará los cambios y actualizará automáticamente el sitio.