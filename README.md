# 🚀 Guía de Despliegue de Rancher en un Cluster RKE

Este documento describe el procedimiento paso a paso para desplegar **Rancher** sobre un **cluster RKE** conformado por **3 máquinas virtuales**:  
- **1 Manager**  
- **2 Workers**

> **Nota:** Es importante **deshabilitar la memoria swap** en los nodos para que Kubernetes pueda programar correctamente los *workloads* en otros nodos cuando se agote la memoria.

---

## 📋 Requisitos Previos

- 3 VMs con conectividad entre sí.
- Acceso SSH entre las VMs.
- Puertos **80** y **443** abiertos para el acceso a Rancher.
- Versiones utilizadas en este procedimiento:
  - **Docker client/server:** 20.10.12  
  - **Kubectl client:** 1.23  
  - **Kubectl server:** 1.21  
  - **RKE:** 1.3.4  
  - **Rancher:** 2.6  
  - **Helm:** 3.7.2  
  - **cert-manager:** 1.5.1  

---

## 🛠 Paso a Paso

### 1️⃣ Instalar Docker en las VMs
[Guía oficial de instalación](https://docs.docker.com/engine/install/ubuntu/)

```bash
apt-get update

apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```

### 2️⃣ Verificar Conectividad entre Nodos

* Probar conectividad entre **workers** y **manager**:

```bash
    ping <IP_ADDRESS_DEL_OTRO_NODO>
```
* Copiar las llaves SSH a ~/.ssh y ajustar permisos:

```bash
mkdir -p ~/.ssh
# Copia tus claves aquí (id_rsa, id_rsa.pub, config, known_hosts, etc.)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/config
chmod 644 ~/.ssh/known_hosts
```

### 3️⃣ Revisar Prerrequisitos de las VMs
    * Requisitos del sistema y compatibilidad: revisar guía oficial de Rancher.
    https://rancher.com/docs/rancher/v2.6/en/installation/requirements/

    * Deshabilitar swap (recomendado para Kubernetes):¨

```bash
sudo swapoff -a
# Deshabilitar swap en reinicios
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
```

### 4️⃣ Configurar Parámetros Requeridos por RKE

    Aplicar en todos los nodos:

```bash
# Asegurar que el módulo está cargado
sudo modprobe br_netfilter

# Reglas persistentes
cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF

# Aplicar inmediatamente
sudo sysctl --system
```

### 5️⃣ Instalar kubectl
    https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

### 6️⃣ Instalar Helm
    https://helm.sh/docs/introhelm/install/

### 7️⃣ Instalar RKE
    https://rancher.com/docs/rke/latest/en/installation/

```bash
# Descargar la versión estable (ejemplo v1.3.4)
curl -LO https://github.com/rancher/rke/releases/download/v1.3.4/rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke
sudo chmod +x /usr/local/bin/rke

# Verificar
rke --version
```

### 8️⃣ Crear el Archivo de Configuración del Cluster
    https://rancher.com/docs/rke/latest/en/installation/

```bash
# Interactivo
rke config

# O bien con nombre específico
rke config --name cluster.yml
```

### 9️⃣ Desplegar Kubernetes con RKE
    
```bash
rke up
```
La última línea debe indicar:

```bash
Finished building Kubernetes cluster successfully
```

### 🔟 Exportar Configuración de RKE (Kubeconfig)
    https://rancher.com/docs/rancher/v2.6/en/installation/resources/k8s-tutorials/ha-rke/

```bash
export KUBECONFIG=./kube_config_cluster.yml
```    
> Tu Kubeconfig para conectarte al cluster

### 1️⃣1️⃣ Agregar el Repositorio Helm de Rancher
    https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
``` 
    
### 1️⃣2️⃣ Crear Namespace para Rancher

```bash        
kubectl create namespace cattle-system
```

### 1️⃣3️⃣ Instalar cert-manager
    https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/

```bash        
# 1) Instalar CRDs
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml

# 2) Agregar repositorio Jetstack
helm repo add jetstack https://charts.jetstack.io

# 3) Actualizar repos
helm repo update

# 4) Instalar cert-manager
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.5.1

# 5) Verificar instalación
kubectl get pods --namespace cert-manager
```

### 1️⃣4️⃣ Instalar Rancher con Helm

```bash
helm install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --set hostname=testcluster.local \
    --set bootstrapPassword=admin
```
> Nota: Ajusta hostname a un FQDN accesible públicamente (o configúralo en /etc/hosts para pruebas).

### 1️⃣5️⃣ Verificar Despliegue de Rancher

```sh
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get deploy rancher
kubectl -n cattle-system get pods -o wide
```

### 1️⃣6️⃣ Acceder a la Interfaz Web de Rancher

    1 - Asegura que los puertos 80 y 443 estén abiertos en el nodo/ingress.

    2 - Si es un entorno de pruebas, añade el hostname etc/hosts:

```sh
echo "<IP_PUBLICA>  testcluster.local" | sudo tee -a /etc/hosts
```

    3 - Accede desde el navegador:
```sh
https://testcluster.local
```

    4 - Inicia sesión con admin y la contraseña definida en bootstrapPassword (cámbiala en el primer ingreso).

---

📚 Referencias

Requisitos de instalación Rancher: https://rancher.com/docs/rancher/v2.6/en/installation/requirements/

RKE Installation: https://rancher.com/docs/rke/latest/en/installation/

Instalar Rancher en Kubernetes: https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/

Kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

Helm: https://helm.sh/docs/intro/install/

---

📦 Versiones Probadas

Docker Engine/CLI: 20.10.12

kubectl (cliente): 1.23

Kubernetes (servidor): 1.21

RKE: 1.3.4

Rancher: 2.6

Helm: 3.7.2

cert-manager: 1.5.1

> Last time tested: 2022
