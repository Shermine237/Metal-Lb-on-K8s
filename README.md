# Metal-Lb-on-K8s
MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.

## Preparation
If you’re using kube-proxy in IPVS mode, since Kubernetes v1.14.2 you have to enable strict ARP mode.
Note, you don’t need this if you’re using kube-router as service-proxy because it is enabling strict ARP by default.
You can achieve this by editing kube-proxy config in current cluster:

```bash
kubectl edit configmap -n kube-system kube-proxy
```
and set:
```bash
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
If you are trying to automate this change, these shell snippets may help you:
```bash
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

## Case 1 : Installation by manifest
To install MetalLB, apply the manifest:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```
## Case 2 : Installation with Helm
You can install MetalLB with Helm by using the Helm chart repository:
```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb
```
## Configuration
MetalLB remains idle until configured. This is accomplished by creating and deploying various resources (IPAddressPool and L2Advertisement)
For example, the following configuration gives MetalLB control over IPs from 192.168.1.240 to 192.168.1.250, and configures Layer 2 mode:
```bash
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```
```bash
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertissement
```
Apply with:
```bash
kubectl apply -f first-pool.ym
kubectl apply -f l2advertissement.ym
```

## Usage
After MetalLB is installed and configured, to expose a service externally, simply create it with spec.type set to LoadBalancer, and MetalLB will do the rest.
Example service to apply (type LoadBalancer):
```bash
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mifos-community
  name: mifos-community
spec:
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 80
  selector:
    app: mifos-community
    tier: frontend
  type: LoadBalancer
```

