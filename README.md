# Instalación de OJS 3.4.0 en Openshift (Image Base ubi9/php-81)

Para esta instalación se tomó como guia la documentación [PKP Docs](https://
docs.pkp.sfu.ca/admin-guide/en/)

Objetos de kubernetes

###BuildConfig
```
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: ojs
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs  
spec:
  output:
    to:
      kind: ImageStreamTag
      name: 'ojs:3.4.0'
  strategy:
    type: Docker
    dockerStrategy:
  source:
    type: Dockerfile
    dockerfile: |
      FROM ubi9/php-81
      USER 0
      # Descarga del configo fuente de la pagina oficial 
      RUN wget -q https://pkp.sfu.ca/ojs/download/ojs-3.4.0-3.tar.gz \
        && mkdir -p /tmp/src && tar xf ojs-3.4.0-3.tar.gz --strip-components=1 -C /tmp/src \
        && rm -f *.tar.gz 

      RUN chown -R 1001:0 /tmp/src
      USER 1001

      # Instalación de dependencias
      RUN /usr/libexec/s2i/assemble

      CMD /usr/libexec/s2i/run
  runPolicy: Serial
```

### ImageStream

```
kind: ImageStream
apiVersion: image.openshift.io/v1
metadata:
  name: ojs
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
spec:
  lookupPolicy:
    local: true
  tags:
  - name: "3.4.0"
```

### Deployment
```
kind: Deployment
apiVersion: apps/v1
metadata:
  annotations:
    image.openshift.io/triggers: >-
      [{"from":{"kind":"ImageStreamTag","name":"ojs:3.4.0"},"fieldPath":"spec.template.spec.containers[?(@.name==\"ojs\")].image"},{"from":{"kind":"ImageStreamTag","name":"ojs:3.4.0"},"fieldPath":"spec.template.spec.initContainers[?(@.name==\"ojs-init\")].image"}]
  name: ojs
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ojs
      app.kubernetes.io/component: ojs
      app.kubernetes.io/instance: ojs        
  template:
    metadata:
      annotations:
        alpha.image.policy.openshift.io/resolve-names: '*'    
      labels:
        app: ojs
        app.kubernetes.io/component: ojs
        app.kubernetes.io/instance: ojs        
    spec:
      volumes:
        - name: ojs-pvc
          persistentVolumeClaim:
            claimName: ojs-pvc
      initContainers:
        - name: ojs-init
          image: ' '
          command: ['sh', '-c', 'if [ -d /tmp/volume/src/ ]; then echo "OJS is installed" && cat dbscripts/xml/version.xml | grep release; else echo "Copy OJS files to PV..."  && mkdir /tmp/volume/files && cp -r /opt/app-root/* /tmp/volume/; fi']
          volumeMounts:
            - name: ojs-pvc
              mountPath: /tmp/volume                 
      containers:
        - name: ojs
          image: ' '
          volumeMounts:
            - name: ojs-pvc
              mountPath: /opt/app-root/      
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
```

### PersistentVolumeClaim

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ojs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ocs-storagecluster-cephfs
```

### Service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
  name: ojs
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: ojs
  type: ClusterIP
```

### Service NodePort

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
  name: ojs-nodeport
spec:
  ports:
  - nodePort: 30370
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
  type: NodePort
```

### Route (Opcional)

```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: ojs
    app.kubernetes.io/component: ojs
    app.kubernetes.io/instance: ojs
  name: ojs
spec:
  port:
    targetPort: 8080
  to:
    kind: Service
    name: ojs
    weight: 100
  wildcardPolicy: None
```
### Pasos para el despliegue en el cluster

Ingresar al directorio donde se encuentran los archivos yaml's

Ejecutar los siguientes comandos:

`oc new-project openjournal-dev`

`oc apply -f .`

`oc start-build ojs`

#### Una vez finalizado el buid y el pod se encuentre en estado "Running" acceder a la aplicación "http://node-ip:node-port" y continuar con el proceso de instalación a travez de la web.

**Base de Datos Mysql 8** (Parametros de ejemplo)

- **Database driver:** mysqli (or "mysql" if your php is lower than 7.3)
- **Host:** mysql 
- **Username:** ojs
- **Password:** ojs
- **Database name:** ojs



### Ingresar al contenedor por la terminal web o cli y editar el archivo config.inc.php ubicado en /opt/app-root/src
```
; This check will invalidate a session if the user’s IP address changes.
; Enabling this option provides some amount of additional security, but may
; cause problems for users behind a proxy farm (e.g., AOL).
session_check_ip = Off

; Generate RESTful URLs using mod_rewrite. This requires the
; rewrite directive to be enabled in your .htaccess or httpd.conf.
; See FAQ for more details.
restful_urls = On
```
### Agregar el archivo .htaccess en el directorio /opt/app-root/src con el siguiente contenido:

.htaccess
```
SetEnvIf X-Forwarded-Proto "https" HTTPS=on
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /
    RewriteRule ^api/v1(._)$ /index.php/api/v1$1 [L,R=307]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(._)$ /index.php/$1 [QSA,L]
</IfModule>
Header add Access-Control-Allow-Origin "dncp.edu.py"
```

### Reiniciar el pod para aplicar los cambios

### El volumen es montado en el directorio /opt/app-root

| Directorio  | Descripción |
|--------------------|--------------------------------------|
| /opt/app-root/src  | Archivos de la aplicación (público)  |
| /opt/app-root/files | Directorio para subir los archivos (privado)  |
