# Proyecto Final Propospad Programación Avanzada Web

## Estudiantes:
- Francini Fernandez Zuñiga 
- Juan Pablo Mayorga Leal
- David Rodriguez Arias 


## Topología:

![alt text](https://github.com/david11alonso/ProyectoFinalKubernetes/blob/main/ProyectoFInal.jpg)


###### Enlances de referencia:
- Citrix-Kubernetes-Monitoreo:https://developer-docs.citrix.com/projects/citrix-observability-exporter/en/latest/deploy-coe-with-es/
- Linode-Kubernetes: https://www.linode.com/docs/guides/getting-started-with-load-balancing-on-a-lke-cluster/



###### Configuración Backend
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-coe
  labels:
    app: backend-coe
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-coe
  template:
    metadata:
      labels:
        app: backend-coe
    spec:
      containers:
      - name: private-reg-container
        image:  david11alonso/prograavanzadaweb:backend
        imagePullPolicy: "Always"
      imagePullSecrets:
      - name: regcred
---
apiVersion: v1
kind: Service
metadata:
  name: backend-coe
  labels:
    app: backend-coe
spec:
  type: NodePort
  ports:
  - name: backend-coe-80
    port: 80
  selector:
    app: backend-coe
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: backend-coe-ingress
  annotations:
   ingress.citrix.com/insecure-termination: "allow"
   kubernetes.io/ingress.class: "backend-coe-ingress"
   ingress.citrix.com/analyticsprofile: '{"webinsight": {"httpurl":"ENABLED", "httpuseragent":"ENABLED", "httphost":"ENABLED", "httpmethod":"ENABLED", "httpcontenttype":"ENABLED"}, "tcpinsight": {"tcpBurstReporting":"DISABLED"}}'
spec:
      rules:
       - http:
          paths:
          - path: /
            backend:
              serviceName: backend-coe
              servicePort: 80

```
