# Control de Acceso Basado en Roles (RBAC) Lab 2 - CKA Course

Este laboratorio está diseñado para practicar la configuración de permisos RBAC en Kubernetes, una de las habilidades clave para la certificación CKA. Role-Based Access Control (RBAC) permite controlar el acceso a los recursos del clúster mediante la asignación de permisos a roles, que luego se aplican a usuarios o ServiceAccounts. En este laboratorio, aprenderás a crear y asignar permisos específicos a través de roles y bindings en diferentes namespaces, asegurando un acceso controlado a los recursos en un clúster de Kubernetes.

Sigue cada sección en el orden indicado para completar con éxito el laboratorio y afianzar tus conocimientos sobre RBAC en Kubernetes.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Objetivos

1. **Configurar namespaces y ServiceAccounts**: Crear namespaces dedicados y verificar la existencia de ServiceAccounts predeterminadas.
2. **Definir roles y permisos específicos**: Crear un Role que permita el acceso controlado a Pods y ConfigMaps en un namespace específico.
3. **Aplicar bindings para asociar roles a ServiceAccounts**: Usar RoleBindings para asociar roles a ServiceAccounts en un namespace específico.
4. **Configurar un ClusterRole para permisos a nivel de clúster**: Crear y asignar un ClusterRole que permita leer Secrets en todo el clúster.
5. **Asociar roles de clúster a cuentas específicas mediante ClusterRoleBindings**: Usar ClusterRoleBindings para asignar permisos de clúster a ServiceAccounts en namespaces específicos.
6. **Verificar permisos y limitaciones mediante pruebas prácticas**: Desplegar pods y realizar solicitudes HTTP para verificar los permisos aplicados, asegurando que las ServiceAccounts tengan acceso limitado a los recursos definidos.

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
   git clone https://github.com/arol-dev/kubernetes-cka-rbac-lab2
   cd kubernetes-cka-backup-recovery-rbac-lab2
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

```bash
kubectl create namespace rbac-dev
```

Verifica que se haya generado automáticamente una ServiceAccount por defecto en el namespace `rbac-dev`:

```bash
kubectl get serviceaccount -n rbac-dev
```


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
```

2. Aplica el Role al clúster:

```bash
kubectl apply -f role-configmap-pod.yaml
```

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
```

2. Aplica el RoleBinding:

```bash
kubectl apply -f rolebinding-configmap-pod.yaml
```

Este RoleBinding vincula el Role `configmap-pod-manager` con la cuenta ServiceAccount `default` en el namespace `rbac-dev`.

### Paso 6: Configuración de Roles y ClusterRoleBindings en `rbac-qa`

1. Crear el Namespace `rbac-qa`

Ejecuta el siguiente comando para crear el namespace `rbac-qa`:

```bash
kubectl create namespace rbac-qa
```

2. Verifica que se haya generado una ServiceAccount por defecto en el namespace `rbac-qa`:

```bash
kubectl get serviceaccount -n rbac-qa
```

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
```

2. Aplica el ClusterRole al clúster:

```bash
kubectl apply -f clusterrole-secret.yaml
```

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
```

2. Aplica el ClusterRoleBinding:

```bash
kubectl apply -f clusterrolebinding-secret.yaml
```

Este ClusterRoleBinding vincula el ClusterRole `secret-reader` con la ServiceAccount `default` en el namespace `rbac-qa`.

### Paso 9: Verificación de Permisos en `rbac-dev`

1. Desplegar un Pod y Verificar Permisos en `rbac-dev`

Despliega un pod usando la imagen `luksa/kubectl-proxy` en el namespace `rbac-dev`:

```bash
kubectl run test --image=luksa/kubectl-proxy -n rbac-dev
```

Confirma que el pod se ha desplegado correctamente:

```bash
kubectl get pods -n rbac-dev
```

2. Conéctate al pod:

```bash
kubectl exec -it test -n rbac-dev -- sh
```

3. Ejecuta una solicitud HTTP para verificar los pods en el namespace `rbac-dev`:

```bash
curl localhost:8001/api/v1/namespaces/rbac-dev/pods
```

4. Para probar la creación de un nuevo pod, crea un archivo `pod.yaml` con una definición básica de un pod de Nginx. Luego ejecuta la solicitud POST:

```bash
curl -X POST --data-binary "@pod.yaml" -H 'Content-Type: application/yaml' localhost:8001/api/v1/namespaces/rbac-dev/pods
```

5. Intenta eliminar el pod y verifica que se reciba un error 403 "Forbidden", lo que indica que la cuenta de servicio en `rbac-dev` no tiene permisos para eliminar pods:

```bash
curl -X DELETE localhost:8001/api/v1/namespaces/rbac-dev/pods/nginx-basic
```

6. Verifica que el pod `nginx-basic` existe en el namespace `rbac-dev`:

```bash
kubectl get pods -n rbac-dev
```

### Paso 10: Verificación de Permisos en `rbac-qa`

1. Desplegar un Pod y Verificar Permisos en `rbac-qa`

Despliega un pod usando la imagen `luksa/kubectl-proxy` en el namespace `rbac-qa`:

```bash
kubectl run test --image=luksa/kubectl-proxy -n rbac-qa
```

Confirma que el pod se ha desplegado correctamente:

```bash
kubectl get pods -n rbac-qa
```

2. Conéctate al pod:

```bash
kubectl exec -it test -n rbac-qa -- sh
```

3. Para verificar los permisos asociados con el ClusterRole `secret-reader`, ejecuta la siguiente solicitud HTTP para obtener los secrets en el clúster:

```bash
curl localhost:8001/api/v1/secrets
```

Deberías ver la lista de secrets definidos en el clúster de Kubernetes, lo cual confirma que el ClusterRole permite el acceso a nivel global para leer secrets.