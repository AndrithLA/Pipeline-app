# Práctica Paso a Paso - Publicación de Imagen con Jenkins Pipeline
### (Variante Docker Hub)

## Requisitos Previos

**Infraestructura Jenkins**
- Jenkins master configurado
- Jenkins agent con Docker instalado
- Plugins necesarios: Docker Pipeline, GitHub Integration, Pipeline: Stage View

**Credenciales**
- Cuenta de Docker Hub con un Access Token (permisos Read & Write)
- GitHub Personal Access Token (opcional, solo si tu repo es privado)

---

## PASO 1: Configurar Credenciales en Jenkins

### 1.1 Agregar Docker Hub Credentials

1. Ve a `Dashboard → Manage Jenkins → Credentials → System → Global credentials (unrestricted)`
2. Haz clic en **Add Credentials**
3. Configura:
   - **Kind**: Username with password
   - **Username**: tu usuario de Docker Hub
   - **Password**: tu Access Token de Docker Hub (no tu contraseña)
   - **ID**: `docker-hub-credentials`
   - **Description**: Docker Hub credentials para publicar imágenes

### 1.2 (Opcional) GitHub Token, solo si necesitas clonar un repo privado

1. **Kind**: Secret text
2. **Secret**: tu token de GitHub
3. **ID**: `github-token`

---

## PASO 2: Estructura del Proyecto

```
mi-proyecto/
├── Jenkinsfile
├── Dockerfile
├── package.json
├── src/
│   └── index.js
└── tests/
    └── test.js
```

Todo esto ya está generado en la carpeta adjunta. El Dockerfile es:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

---

## PASO 3: Jenkinsfile - Pipeline Declarativo

El `Jenkinsfile` ya está creado en la raíz del proyecto, con `REGISTRY = 'docker.io'`
y el stage `Push to Docker Hub` usando las credenciales `docker-hub-credentials`
(equivalente al PASO 4.2 del documento original, integrado directo en el
pipeline principal en vez de como bloque aparte).

Antes de usarlo, edita esta línea del `Jenkinsfile`:

```groovy
IMAGE_NAME = 'tu-usuario-dockerhub/mi-app'
```

y reemplaza `tu-usuario-dockerhub` por tu usuario real de Docker Hub.

---

## PASO 4: Configuración de Credenciales en Jenkinsfile

### 4.1 Usar Credenciales de Jenkins (versión original del PDF, con GHCR)

Esto es lo que dice el PDF tal cual, usando GitHub Container Registry.
Modifica el stage de publicación para usar credenciales de Jenkins:

```groovy
stage('Push to Registry') {
    steps {
        withCredentials([
            string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')
        ]) {
            sh '''
                echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER --password-stdin
                docker push ${IMAGE_TAG_COMMIT}
                docker push ${IMAGE_TAG_LATEST}
            '''
        }
    }
}
```

Nota: aquí `$GITHUB_USER` tendría que venir de otra credencial o variable de
entorno adicional (el PDF no la define explícitamente en este bloque).

### 4.2 Para Docker Hub (la variante que estamos usando en este proyecto)

```groovy
stage('Push to Docker Hub') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'docker-hub-credentials',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )
        ]) {
            sh '''
                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                docker push ${IMAGE_TAG_COMMIT}
                docker push ${IMAGE_TAG_LATEST}
            '''
        }
    }
}
```

**Esta es la versión (4.2) que ya está implementada en el `Jenkinsfile` de este
proyecto**, porque escogiste Docker Hub como registro. La 4.1 la dejo solo
como referencia por si más adelante quieres comparar contra GHCR o cambiarte.

---

## PASO 5: Versión con Pipeline Scripted (Opcional)

Alternativa usando pipeline scripted, adaptada a Docker Hub:

```groovy
node {
    def imageName = 'tu-usuario-dockerhub/mi-app'
    def commitSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    def image = "${imageName}:${commitSha}"
    def imageLatest = "${imageName}:latest"

    stage('Checkout') {
        checkout scm
    }

    stage('Test') {
        sh 'npm ci'
        sh 'npm test'
    }

    stage('Build Image') {
        docker.build(image)
    }

    stage('Push Image') {
        withCredentials([
            usernamePassword(
                credentialsId: 'docker-hub-credentials',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )
        ]) {
            sh '''
                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                docker push ${image}
                docker tag ${image} ${imageLatest}
                docker push ${imageLatest}
            '''
        }
    }

    stage('Cleanup') {
        sh 'docker system prune -f'
    }
}
```

---

## PASO 6: Configurar Jenkins Job

### 6.1 Crear Pipeline Job

1. `Dashboard → New Item`
2. Nombre: `mi-app-pipeline`
3. Tipo: **Pipeline**
4. Configuración:
   - **Definition**: Pipeline script from SCM
   - **SCM**: Git
   - **Repository URL**: `https://github.com/tu-usuario/mi-proyecto.git`
   - **Credentials**: (agrega credenciales de GitHub si el repo es privado)
   - **Branch**: `*/main`
   - **Script Path**: `Jenkinsfile`

### 6.2 Configurar Webhook en GitHub

Para trigger automático:
1. En GitHub: `Settings → Webhooks → Add webhook`
2. **Payload URL**: `http://jenkins-server:8080/github-webhook/`
3. **Content type**: `application/json`
4. **Events**: Just the push event

---

## PASO 7: Ejecutar y Verificar (10 min)

### 7.1 Ejecutar Manualmente

1. Ir al job creado
2. Click en **Build Now**
3. Monitorear la consola de salida

### 7.2 Verificar Publicación

```bash
# Verificar imagen publicada en Docker Hub
docker pull tu-usuario-dockerhub/mi-app:latest
docker pull tu-usuario-dockerhub/mi-app:$(git rev-parse --short HEAD)

# Verificar tags
docker image inspect tu-usuario-dockerhub/mi-app:latest
```

También puedes verla entrando a tu perfil en hub.docker.com → pestaña **Repositories**.

---

## PASO 8: Diagrama del Pipeline Jenkins

```
┌─────────────────────────────────────────────────────────┐
│                  JENKINS PIPELINE                        │
├─────────────────────────────────────────────────────────┤
│                                                            │
│  [GitHub Push] → [Trigger Webhook]                        │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Prepare    │                                    │
│  │ - Checkout SCM     │                                   │
│  │ - Docker version   │                                   │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Install     │                                   │
│  │ - npm ci            │                                  │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Test         │                                  │
│  │ - npm test           │                                 │
│  │ - JUnit reports      │                                 │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Build         │                                 │
│  │ - docker build        │                                │
│  │ - Tag: {commit}       │                                │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Push (Docker Hub) │                              │
│  │ - Login Docker Hub         │                            │
│  │ - Push {commit}             │                           │
│  │ - Push latest                │                          │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  ┌──────────────────┐                                     │
│  │ STAGE: Verify        │                                 │
│  │ - Image exists         │                                │
│  └──────────────────┘                                     │
│         ↓                                                 │
│  [Post Actions]                                            │
│  - Email notifications                                     │
│  - Cleanup resources                                       │
│                                                             │
└─────────────────────────────────────────────────────────┘
```

---

## Solución de Problemas Comunes en Jenkins

**Error: "docker: command not found"**
```groovy
// Solución: Instalar Docker en el agente o usar nodo con Docker
agent {
    docker {
        image 'docker:latest'
        args '-v /var/run/docker.sock:/var/run/docker.sock'
    }
}
```

**Error: "denied: requested access to the resource is denied"**
```groovy
// Solución: Verificar usuario, token y permisos del token en Docker Hub
withCredentials([
    usernamePassword(
        credentialsId: 'docker-hub-credentials',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )
]) {
    sh '''
        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
    '''
}
```

**Error: "No such image"**
```groovy
// Solución: Asegurar el nombre completo de la imagen
def fullImageName = "${imageName}:${tag}"
docker.build(fullImageName)
```

---

## Entregables Finales

**Checklist de Verificación:**
- [ ] Jenkins configurado con plugins necesarios
- [ ] Credenciales configuradas (`docker-hub-credentials`)
- [ ] Jenkinsfile creado con stages:
  - [ ] Prepare
  - [ ] Install Dependencies
  - [ ] Test
  - [ ] Build Docker Image
  - [ ] Push to Docker Hub
  - [ ] Verify Published Image
- [ ] Pipeline ejecutado exitosamente
- [ ] Imagen publicada en Docker Hub
- [ ] Tags verificados: commit SHA, latest, build timestamp
- [ ] Diagrama del pipeline completo
- [ ] Notificaciones configuradas
