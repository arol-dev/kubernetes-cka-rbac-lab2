# Kubernetes RBAC Lab for CKA Certification

Este laboratorio está diseñado para practicar la configuración de permisos RBAC en Kubernetes. Configuraremos roles y bindings en distintos namespaces y probaremos los permisos asociados.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Objetivos

1. **Comprender los elementos básicos de RBAC**: roles, bindings, permisos y namespaces.
2. **Configurar certificados X.509** para la autenticación del usuario operador.
3. **Crear y asociar roles específicos** para otorgar permisos limitados sobre los recursos.
4. **Practicar el flujo de trabajo de configuración** de permisos de acceso, verificación de permisos y modificación de roles.

## Contenido del Repositorio

Este repositorio incluye:

- Una carpeta `scripts` con dos scripts que proporcionan soporte durante el laboratorio.
- Un fichero `Vagrantfile` que permite automatizar el despliegue de tres VMs en VirtualBox.

Las VMs consisten en:

- 1 nodo master.
- 2 nodos worker.

### Paso 1: Despliegue de las VMs

1. Clona el repositorio en tu entorno local:

   ```bash
   git clone https://github.com/arol-dev/kubernetes-cka-rbac.git
   cd kubernetes-cka-backup-recovery-rbac
   ```

2. Dentro del repositorio, ejecuta el siguiente comando para desplegar las VMs:

   ```bash
   vagrant up
   ```

   Esto comenzará a desplegar tres VMs en VirtualBox: un nodo master y dos worker nodes. Espera unos minutos para que el proceso termine.

3. Verifica el estado de las VMs con:

   ```bash
   vagrant status
   ```

   Asegúrate de que las tres máquinas estén en estado `running`.

4. Obtén la configuración SSH para conectarte a las máquinas:

   ```bash
   vagrant ssh-config
   ```

   Guarda los detalles proporcionados, ya que los necesitarás en el siguiente paso.

### Paso 2: Conectar a las VMs con MobaXterm

1. Abre **MobaXterm** y utiliza la configuración SSH obtenida anteriormente para conectarte a las tres máquinas.
   - No se requiere un usuario específico, deja el campo vacío.
   - Si se te solicita usuario o contraseña, utiliza la cadena `vagrant`.

### Paso 3: Configuración Inicial del Namespace `rbac-dev`

1. Crear el Namespace `rbac-dev`

Ejecuta el siguiente comando para crear el namespace `rbac-dev`:

```bash
kubectl create namespace rbac-dev
```{{exec}}

Verifica que se haya generado automáticamente una ServiceAccount por defecto en el namespace `rbac-dev`:

```bash
kubectl get serviceaccount -n rbac-dev
```{{exec}}

### Paso 4: Configuración de Roles y RoleBindings en `rbac-dev`

1. Crear un Role para Gestionar Pods y ConfigMaps

Crea un archivo YAML llamado `role-configmap-pod.yaml` con el siguiente contenido:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-dev
  name: configmap-pod-manager
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["create", "get", "list"]
```{{copy}}

2. Aplica el Role al clúster:

```bash
kubectl apply -f role-configmap-pod.yaml
```{{exec}}

### Paso 5: Crear un RoleBinding para Vincular el Role al ServiceAccount `default`

1. Crea un archivo YAML llamado `rolebinding-configmap-pod.yaml` con el siguiente contenido:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-pod-binding
  namespace: rbac-dev
subjects:
- kind: ServiceAccount
  name: default
  namespace: rbac-dev
roleRef:
  kind: Role
  name: configmap-pod-manager
  apiGroup: rbac.authorization.k8s.io
```{{copy}}

2. Aplica el RoleBinding:

```bash
kubectl apply -f rolebinding-configmap-pod.yaml
```{{exec}}

Este RoleBinding vincula el Role `configmap-pod-manager` con la cuenta ServiceAccount `default` en el namespace `rbac-dev`.

### Paso 6: Configuración de Roles y ClusterRoleBindings en `rbac-qa`

1. Crear el Namespace `rbac-qa`

Ejecuta el siguiente comando para crear el namespace `rbac-qa`:

```bash
kubectl create namespace rbac-qa
```{{exec}}

2. Verifica que se haya generado una ServiceAccount por defecto en el namespace `rbac-qa`:

```bash
kubectl get serviceaccount -n rbac-qa
```{{exec}}

### Paso 7: Crear un ClusterRole para Leer Secrets en Todo el Clúster

1. Crea un archivo YAML llamado `clusterrole-secret.yaml` con el siguiente contenido:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```{{copy}}

2. Aplica el ClusterRole al clúster:

```bash
kubectl apply -f clusterrole-secret.yaml
```{{exec}}

### Paso 8: Crear un ClusterRoleBinding para Vincular el ClusterRole al ServiceAccount `default` en `rbac-qa`

1. Crea un archivo YAML llamado `clusterrolebinding-secret.yaml` con el siguiente contenido:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: secret-reader-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: rbac-qa
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```{{copy}}

2. Aplica el ClusterRoleBinding:

```bash
kubectl apply -f clusterrolebinding-secret.yaml
```{{exec}}

Este ClusterRoleBinding vincula el ClusterRole `secret-reader` con la ServiceAccount `default` en el namespace `rbac-qa`.

### Paso 9: Verificación de Permisos en `rbac-dev`

1. Desplegar un Pod y Verificar Permisos en `rbac-dev`

Despliega un pod usando la imagen `luksa/kubectl-proxy` en el namespace `rbac-dev`:

```bash
kubectl run test --image=luksa/kubectl-proxy -n rbac-dev
```{{exec}}

Confirma que el pod se ha desplegado correctamente:

```bash
kubectl get pods -n rbac-dev
```{{exec}}

2. Conéctate al pod:

```bash
kubectl exec -it test -n rbac-dev -- sh
```{{exec}}

3. Ejecuta una solicitud HTTP para verificar los pods en el namespace `rbac-dev`:

```bash
curl localhost:8001/api/v1/namespaces/rbac-dev/pods
```{{exec}}

4. Para probar la creación de un nuevo pod, crea un archivo `pod.yaml` con una definición básica de un pod de Nginx. Luego ejecuta la solicitud POST:

```bash
curl -X POST --data-binary "@pod.yaml" -H 'Content-Type: application/yaml' localhost:8001/api/v1/namespaces/rbac-dev/pods
```{{exec}}

5. Intenta eliminar el pod y verifica que se reciba un error 403 "Forbidden", lo que indica que la cuenta de servicio en `rbac-dev` no tiene permisos para eliminar pods:

```bash
curl -X DELETE localhost:8001/api/v1/namespaces/rbac-dev/pods/nginx-basic
```{{exec}}

6. Verifica que el pod `nginx-basic` existe en el namespace `rbac-dev`:

```bash
kubectl get pods -n rbac-dev
```{{exec}}

### Paso 10: Verificación de Permisos en `rbac-qa`

1. Desplegar un Pod y Verificar Permisos en `rbac-qa`

Despliega un pod usando la imagen `luksa/kubectl-proxy` en el namespace `rbac-qa`:

```bash
kubectl run test --image=luksa/kubectl-proxy -n rbac-qa
```{{exec}}

Confirma que el pod se ha desplegado correctamente:

```bash
kubectl get pods -n rbac-qa
```{{exec}}

2. Conéctate al pod:

```bash
kubectl exec -it test -n rbac-qa -- sh
```{{exec}}

3. Para verificar los permisos asociados con el ClusterRole `secret-reader`, ejecuta la siguiente solicitud HTTP para obtener los secrets en el clúster:

```bash
curl localhost:8001/api/v1/secrets
```{{exec}}

Deberías ver la lista de secrets definidos en el clúster de Kubernetes, lo cual confirma que el ClusterRole permite el acceso a nivel global para leer secrets.

## Conclusión

Has configurado exitosamente roles, rolebindings, clusterroles, y clusterrolebindings en Kubernetes y verificado los permisos en cada namespace.