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



###### Configuración Backend --DeploymentBackendMetrics.yaml--
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
###### Configuración CPX --DeploymentCPXMetrics.yaml--
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cpx-ingress-k8s-role
rules:
  - apiGroups: [""]
    resources: ["endpoints", "ingresses", "pods", "secrets", "nodes", "routes", "namespaces", "configmaps"]
    verbs: ["get", "list", "watch"]
  # services/status is needed to update the loadbalancer IP in service status for integrating
  # service of type LoadBalancer with external-dns
  - apiGroups: [""]
    resources: ["services/status"]
    verbs: ["patch"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses/status"]
    verbs: ["patch"]
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["citrix.com"]
    resources: ["rewritepolicies", "continuousdeployments", "authpolicies", "ratelimits", "listeners", "httproutes", "wafs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["citrix.com"]
    resources: ["rewritepolicies/status", "continuousdeployments/status", "ratelimits/status", "authpolicies/status", "listeners/status", "httproutes/status", "wafs/status"]
    verbs: ["get", "list", "patch"]
  - apiGroups: ["citrix.com"]
    resources: ["vips"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cpx-ingress-k8s-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cpx-ingress-k8s-role
subjects:
- kind: ServiceAccount
  name: cpx-ingress-k8s-role
  namespace: universidad

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: cpx-ingress-k8s-role
  namespace: universidad

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpx-ingress-es
spec:
  selector:
    matchLabels:
      app: cpx-ingress-es
  replicas: 1
  template:
    metadata:
      name: cpx-ingress-es
      labels:
        app: cpx-ingress-es
      annotations:
    spec:
      serviceAccountName: cpx-ingress-k8s-role
      containers:
        - name: cpx-ingress-es
          image: "quay.io/citrix/citrix-k8s-cpx-ingress:13.0-64.35"
          volumeMounts:
            - mountPath: /var/deviceinfo
              name: shared-data
          securityContext:
             privileged: true
          env:
          - name: "EULA"
            value: "yes"
          - name: "KUBERNETES_TASK_ID"
            value: ""
          imagePullPolicy: Always
        # Add cic as a sidecar
        - name: cic
          image: "quay.io/citrix/citrix-k8s-ingress-controller:1.9.9"
          volumeMounts:
            - mountPath: /var/deviceinfo
              name: shared-data
          env:
          - name: "EULA"
            value: "yes"
          - name: "NS_IP"
            value: "127.0.0.1"
          - name: "NS_PROTOCOL"
            value: "HTTP"
          - name: "NS_PORT"
            value: "80"
          - name: "NS_DEPLOYMENT_MODE"
            value: "SIDECAR"
          - name: "NS_ENABLE_MONITORING"
            value: "YES"
          - name: POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
          envFrom:
          - configMapRef:
              name: cic-configmap
          args:
            - --ingress-classes
              backend-coe-ingress
          imagePullPolicy: Always
      volumes:
      - name: shared-data
        emptyDir: {}
          

---
apiVersion: v1
kind: Service
metadata:
  name: cpx-ingress-es
  labels:
    app: cpx-ingress-es
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: cpx-ingress-es
```
###### Configuración COE  --ConfigMapMetrics.yaml DeploymentCOEMetrics.yaml--
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cic-configmap
  namespace: universidad
data:
   NS_ANALYTICS_CONFIG: |
     distributed_tracing:
       enable: 'false'
       samplingrate: 0
     endpoint:
       server: 'coe-es.universidad.svc.cluster.local'
     timeseries: 
       port: 5563 
       metrics:
         enable: 'false'
         mode: 'prometheus' 
       auditlogs:
         enable: 'false'
       events: 
         enable: 'false'
     transactions:
       enable: 'true'
       port: 5557
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coe-config-es
data:
  lstreamd_default.conf: |
    {
        "Endpoints": {
            "ES": {
                "ServerUrl": "elasticsearch.universidad.svc.cluster.local:9200",
                "IndexPrefix":"adc_coe",
                "IndexInterval": "daily",
                "RecordType": {
                    "HTTP": "all",
                    "TCP": "all",
                    "SWG": "all",
                    "VPN": "all",
                    "NGS": "all",
                    "ICA": "all",
                    "APPFW": "none",
                    "BOT": "none",
                    "VIDEOOPT": "none",
                    "BURST_CQA": "none",
                    "SLA": "none",
                    "MONGO": "none"
                },
                "ProcessAlways": "no",
                "ProcessYieldTimeOut": "500",
                "MaxConnections": "512",
                "ElkMaxSendBuffersPerSec": "64",
                "JsonFileDump": "no"
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coe-es
  labels: 
    app: coe-es
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coe-es
  template:
    metadata:
      name: coe-es
      labels:
        app: coe-es
    spec:
      containers:
        - name: coe-es
          image: "quay.io/citrix/citrix-observability-exporter:1.2.001"
          imagePullPolicy: Always
          ports:
            - containerPort: 5557
              name: lstream
          volumeMounts:
            - name: lstreamd-config-es
              mountPath: /var/logproxy/lstreamd/conf/lstreamd_default.conf
              subPath: lstreamd_default.conf
            - name: core-data
              mountPath: /cores/
      volumes:
        - name: lstreamd-config-es
          configMap:
            name: coe-config-es
        - name: core-data
          emptyDir: {}
---
# Citrix-observability-exporter headless service  
apiVersion: v1
kind: Service
metadata:
  name: coe-es
  labels:
    app: coe-es
spec:
  clusterIP: None
  ports:
    - port: 5557
      protocol: TCP
  selector:
      app: coe-es
---
# Citrix-observability-exporter NodePort service
apiVersion: v1
kind: Service
metadata:
  name: coe-es-nodeport
  labels:
    app: coe-es
spec:
  type: NodePort
  ports:
    - port: 5557
      protocol: TCP
  selector: 
      app: coe-es
```
###### Configuración ELK/Kibana  --elasticsearch.yaml kibana.yaml--
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels: 
      app: elasticsearch
  replicas: 1
  template:
    metadata:
      name: elasticsearch
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image:  docker.elastic.co/elasticsearch/elasticsearch:7.8.0
        env:
         - name: "discovery.type"
           value: "single-node"
        ports:
        - name: es-9200
          containerPort: 9200
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  labels:
    app: elasticsearch
spec:
  type: NodePort
  ports:
  - name: es-9200
    port: 9200
  selector:
    app: elasticsearch

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  selector:
    matchLabels: 
      app: kibana
  replicas: 1
  template:
    metadata:
      name: kibana
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.8.0
        env:
         - name: "ELASTICSEARCH_URL"
           value: "http://elasticsearch.universidad.svc.cluster.local:9200"
        ports:
        - name: kibana-5601
          containerPort: 5601
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  labels: 
    app: kibana
spec:
  type: NodePort
  ports:
  - name: kibana-5601
    port: 5601
  selector:
    app: kibana
---

```
###### Configuración Frontend  --DeploymentFrontEnd.yaml--
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: private-reg-container
        image:  david11alonso/prograavanzadaweb:frontend
        imagePullPolicy: "Always"
      imagePullSecrets:
      - name: regcred
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  type: LoadBalancer
  ports:
  - name: frontend-80
    port: 80
  selector:
    app: frontend
```
###### Gráficos Kibana  --ModelKibana.ndjson--
```json
{"attributes":{"fieldFormatMap":"{\"HttpContentType\":{\"id\":\"string\",\"params\":{\"parsedUrl\":{\"origin\":\"http://10.102.40.41:30633\",\"pathname\":\"/app/kibana\",\"basePath\":\"\"}}},\"ServerProcessingTime\":{\"id\":\"duration\",\"params\":{\"parsedUrl\":{\"origin\":\"http://10.102.40.41:30633\",\"pathname\":\"/app/kibana\",\"basePath\":\"\"},\"inputFormat\":\"microseconds\",\"outputFormat\":\"asSeconds\"}},\"httpReqUrl\":{\"id\":\"url\",\"params\":{\"parsedUrl\":{\"origin\":\"http://10.102.40.41:30633\",\"pathname\":\"/app/kibana\",\"basePath\":\"\"}}}}","fields":"[{\"name\":\"_id\",\"type\":\"string\",\"esTypes\":[\"_id\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":false},{\"name\":\"_index\",\"type\":\"string\",\"esTypes\":[\"_index\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":false},{\"name\":\"_score\",\"type\":\"number\",\"count\":0,\"scripted\":false,\"searchable\":false,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"_source\",\"type\":\"_source\",\"esTypes\":[\"_source\"],\"count\":0,\"scripted\":false,\"searchable\":false,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"_type\",\"type\":\"string\",\"esTypes\":[\"_type\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":false},{\"name\":\"actualtemplatecode\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"appName\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"appName.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"appName\"}}},{\"name\":\"appNameVserverLs\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"appNameVserverLs.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"appNameVserverLs\"}}},{\"name\":\"backendSvrDstIpv4Address\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"backendSvrIpv4Address\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"clientMss\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"clntTcpJitter\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"cltFlowFlagsRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"cltFlowFlagsTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"cltTcpFlagsRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"cltTcpFlagsTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"flowFlagsRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpContentType\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"httpContentType.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"httpContentType\"}}},{\"name\":\"httpReqForwFB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpReqForwLB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpReqMethod\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"httpReqMethod.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"httpReqMethod\"}}},{\"name\":\"httpReqRcvFB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpReqRcvLB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpReqUrl\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"httpReqUrl.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"httpReqUrl\"}}},{\"name\":\"httpReqUserAgent\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"httpReqUserAgent.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"httpReqUserAgent\"}}},{\"name\":\"httpResForwFB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpResForwLB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpResRcvFB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpResRcvLB\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpRspLen\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpRspStatus\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"httpTransEndTime\",\"type\":\"date\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"ingressInterfaceClient\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"observationPointId\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"recType\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"recType.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"recType\"}}},{\"name\":\"reqTimestamp\",\"type\":\"date\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"srvFlowFlagsRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"srvFlowFlagsTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"srvrTcpJitter\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"svrTcpFlagsRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"svrTcpFlagsTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"tcpSrvrConnRstCode\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"tracingReqSpanId\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"tracingReqSpanId.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"tracingReqSpanId\"}}},{\"name\":\"tracingRspSpanId\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"tracingRspSpanId.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"tracingRspSpanId\"}}},{\"name\":\"tracingTraceId\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":false,\"readFromDocValues\":false},{\"name\":\"tracingTraceId.keyword\",\"type\":\"string\",\"esTypes\":[\"keyword\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true,\"subType\":{\"multi\":{\"parent\":\"tracingTraceId\"}}},{\"name\":\"transCltDstIpv4Address\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltDstPort\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltFlowEndUsecRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltFlowEndUsecTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltFlowStartUsecRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltFlowStartUsecTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltIpv4Address\",\"type\":\"string\",\"esTypes\":[\"text\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltRTT\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltSrcPort\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltTotRxOctCnt\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transCltTotTxOctCnt\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transInfo\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSrvDstPort\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSrvSrcPort\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrFlowEndUsecRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrFlowEndUsecTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrFlowStartUsecRx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrFlowStartUsecTx\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrRTT\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrTotRxOctCnt\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transSvrTotTxOctCnt\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"transactionId\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"vlanNumber\",\"type\":\"number\",\"esTypes\":[\"long\"],\"count\":0,\"scripted\":false,\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":true},{\"name\":\"ServerProcessingTime\",\"type\":\"number\",\"count\":1,\"scripted\":true,\"script\":\"if (doc['httpResForwLB'].size() != 0 && doc['httpReqRcvFB'].size() != 0) {\\r\\n  // do something if \\\"item\\\" is missing\\r\\n  return (double)doc['httpResForwLB'].value - (double)doc['httpReqRcvFB'].value ;\\r\\n} else {\\r\\n  return 0;\\r\\n}\",\"lang\":\"painless\",\"searchable\":true,\"aggregatable\":true,\"readFromDocValues\":false}]","title":"adc_coe*"},"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","migrationVersion":{"index-pattern":"7.6.0"},"references":[],"type":"index-pattern","updated_at":"2020-11-09T13:05:05.621Z","version":"WzUzLDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[],\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\"}"},"title":"Total HTTP Requests","uiStateJSON":"{}","version":1,"visState":"{\"title\":\"Total HTTP Requests\",\"type\":\"metric\",\"params\":{\"metric\":{\"percentageMode\":false,\"useRanges\":false,\"colorSchema\":\"Green to Red\",\"metricColorMode\":\"None\",\"colorsRange\":[{\"type\":\"range\",\"from\":0,\"to\":10000}],\"labels\":{\"show\":true},\"invertColors\":false,\"style\":{\"bgFill\":\"#000\",\"bgColor\":false,\"labelColor\":false,\"subText\":\"\",\"fontSize\":60}},\"dimensions\":{\"metrics\":[{\"type\":\"vis_dimension\",\"accessor\":0,\"format\":{\"id\":\"number\",\"params\":{}}}]},\"addTooltip\":true,\"addLegend\":false,\"type\":\"metric\"},\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{\"customLabel\":\"# HTTP requests\"}}]}"},"id":"c6e18cb0-11b3-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzU0LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[],\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\"}"},"title":"HTTP request methods","uiStateJSON":"{}","version":1,"visState":"{\"title\":\"HTTP request methods\",\"type\":\"metric\",\"params\":{\"metric\":{\"percentageMode\":false,\"useRanges\":false,\"colorSchema\":\"Green to Red\",\"metricColorMode\":\"None\",\"colorsRange\":[{\"type\":\"range\",\"from\":0,\"to\":10000}],\"labels\":{\"show\":true},\"invertColors\":false,\"style\":{\"bgFill\":\"#000\",\"bgColor\":false,\"labelColor\":false,\"subText\":\"\",\"fontSize\":60}},\"dimensions\":{\"metrics\":[{\"type\":\"vis_dimension\",\"accessor\":1,\"format\":{\"id\":\"number\",\"params\":{}}}],\"bucket\":{\"type\":\"vis_dimension\",\"accessor\":0,\"format\":{\"id\":\"terms\",\"params\":{\"id\":\"string\",\"otherBucketLabel\":\"Other\",\"missingBucketLabel\":\"Missing\"}}}},\"addTooltip\":true,\"addLegend\":false,\"type\":\"metric\"},\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{\"customLabel\":\"Requests\"}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"group\",\"params\":{\"field\":\"httpReqMethod.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":50,\"otherBucket\":false,\"otherBucketLabel\":\"Other\",\"missingBucket\":false,\"missingBucketLabel\":\"Missing\"}}]}"},"id":"dd7708e0-11b5-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzU1LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[],\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\"}"},"title":"Response code Distribution","uiStateJSON":"{}","version":1,"visState":"{\"title\":\"Response code Distribution\",\"type\":\"pie\",\"params\":{\"type\":\"pie\",\"addTooltip\":true,\"addLegend\":true,\"legendPosition\":\"right\",\"isDonut\":true,\"labels\":{\"show\":false,\"values\":true,\"last_level\":true,\"truncate\":100},\"dimensions\":{\"metric\":{\"accessor\":1,\"format\":{\"id\":\"number\"},\"params\":{},\"aggType\":\"count\"},\"buckets\":[{\"accessor\":0,\"format\":{\"id\":\"terms\",\"params\":{\"id\":\"number\",\"otherBucketLabel\":\"Other\",\"missingBucketLabel\":\"Missing\"}},\"params\":{},\"aggType\":\"terms\"}]}},\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"segment\",\"params\":{\"field\":\"httpRspStatus\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":5,\"otherBucket\":false,\"otherBucketLabel\":\"Other\",\"missingBucket\":false,\"missingBucketLabel\":\"Missing\",\"customLabel\":\"Response codes\"}}]}"},"id":"2238aa80-11a0-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzU2LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[],\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\"}"},"title":"HttpContentType distribution","uiStateJSON":"{}","version":1,"visState":"{\"title\":\"HttpContentType distribution\",\"type\":\"pie\",\"params\":{\"type\":\"pie\",\"addTooltip\":true,\"addLegend\":true,\"legendPosition\":\"right\",\"isDonut\":true,\"labels\":{\"show\":false,\"values\":true,\"last_level\":true,\"truncate\":100},\"dimensions\":{\"metric\":{\"accessor\":0,\"format\":{\"id\":\"number\"},\"params\":{},\"aggType\":\"count\"}}},\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"segment\",\"params\":{\"field\":\"httpContentType.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":5,\"otherBucket\":false,\"otherBucketLabel\":\"Other\",\"missingBucket\":false,\"missingBucketLabel\":\"Missing\"}}]}"},"id":"1328ca80-11b3-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzU3LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\",\"filter\":[]}"},"title":"URL vs ResponseCode","uiStateJSON":"{}","version":1,"visState":"{\"type\":\"histogram\",\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{\"customLabel\":\"Number of Requests\"}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"segment\",\"params\":{\"field\":\"httpReqUrl.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":10,\"otherBucket\":false,\"otherBucketLabel\":\"Other\",\"missingBucket\":true,\"missingBucketLabel\":\"Missing\",\"customLabel\":\"Http Request URL\"}},{\"id\":\"3\",\"enabled\":true,\"type\":\"range\",\"schema\":\"group\",\"params\":{\"field\":\"httpRspStatus\",\"ranges\":[{\"from\":100,\"to\":199},{\"from\":200,\"to\":299},{\"from\":300,\"to\":399},{\"from\":400,\"to\":499},{\"from\":500,\"to\":599}],\"customLabel\":\"Response Codes\"}}],\"params\":{\"type\":\"histogram\",\"grid\":{\"categoryLines\":false},\"categoryAxes\":[{\"id\":\"CategoryAxis-1\",\"type\":\"category\",\"position\":\"left\",\"show\":true,\"style\":{},\"scale\":{\"type\":\"linear\"},\"labels\":{\"show\":true,\"rotate\":0,\"filter\":false,\"truncate\":200},\"title\":{}}],\"valueAxes\":[{\"id\":\"ValueAxis-1\",\"name\":\"LeftAxis-1\",\"type\":\"value\",\"position\":\"bottom\",\"show\":true,\"style\":{},\"scale\":{\"type\":\"linear\",\"mode\":\"normal\"},\"labels\":{\"show\":true,\"rotate\":75,\"filter\":true,\"truncate\":100},\"title\":{\"text\":\"Number of Requests\"}}],\"seriesParams\":[{\"show\":true,\"type\":\"histogram\",\"mode\":\"normal\",\"data\":{\"label\":\"Number of Requests\",\"id\":\"1\"},\"valueAxis\":\"ValueAxis-1\",\"drawLinesBetweenPoints\":true,\"showCircles\":true}],\"addTooltip\":true,\"addLegend\":true,\"legendPosition\":\"right\",\"times\":[],\"addTimeMarker\":false,\"labels\":{},\"dimensions\":{\"x\":{\"accessor\":0,\"format\":{\"id\":\"terms\",\"params\":{\"id\":\"string\",\"otherBucketLabel\":\"Other\",\"missingBucketLabel\":\"Missing\"}},\"params\":{},\"aggType\":\"terms\"},\"y\":[{\"accessor\":2,\"format\":{\"id\":\"number\"},\"params\":{},\"aggType\":\"count\"}],\"series\":[{\"accessor\":1,\"format\":{\"id\":\"range\",\"params\":{\"id\":\"number\"}},\"params\":{},\"aggType\":\"range\"}]},\"thresholdLine\":{\"show\":false,\"value\":10,\"width\":1,\"style\":\"full\",\"color\":\"#E7664C\"}},\"title\":\"URL vs ResponseCode\"}"},"id":"f2cbd110-119e-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:12:02.995Z","version":"WzY5LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\",\"filter\":[]}"},"title":"URL Vs Duration","uiStateJSON":"{}","version":1,"visState":"{\"type\":\"histogram\",\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"avg\",\"schema\":\"metric\",\"params\":{\"field\":\"ServerProcessingTime\",\"customLabel\":\"Duration in seconds\"}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"segment\",\"params\":{\"field\":\"httpReqUrl.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":5,\"otherBucket\":true,\"otherBucketLabel\":\"Other\",\"missingBucket\":true,\"missingBucketLabel\":\"Missing\",\"customLabel\":\"HTTP URL\"}},{\"id\":\"3\",\"enabled\":true,\"type\":\"range\",\"schema\":\"group\",\"params\":{\"field\":\"transSvrRTT\",\"ranges\":[{\"from\":0,\"to\":500},{\"from\":500,\"to\":1000},{\"from\":1000,\"to\":1500},{\"from\":1500,\"to\":2000},{\"from\":2000,\"to\":2500},{\"from\":2500,\"to\":3000},{\"from\":3000}],\"customLabel\":\"Server RTT in ms\"}}],\"params\":{\"addLegend\":true,\"addTimeMarker\":false,\"addTooltip\":true,\"categoryAxes\":[{\"id\":\"CategoryAxis-1\",\"labels\":{\"filter\":true,\"rotate\":75,\"show\":true,\"truncate\":100},\"position\":\"bottom\",\"scale\":{\"type\":\"linear\"},\"show\":true,\"style\":{},\"title\":{},\"type\":\"category\"}],\"dimensions\":{\"series\":[{\"accessor\":1,\"aggType\":\"range\",\"format\":{\"id\":\"range\",\"params\":{\"id\":\"number\"}},\"params\":{}}],\"x\":{\"accessor\":0,\"aggType\":\"terms\",\"format\":{\"id\":\"terms\",\"params\":{\"id\":\"string\",\"missingBucketLabel\":\"Missing\",\"otherBucketLabel\":\"Other\"}},\"params\":{}},\"y\":[{\"accessor\":2,\"aggType\":\"count\",\"format\":{\"id\":\"number\"},\"params\":{}}]},\"grid\":{\"categoryLines\":false},\"labels\":{\"show\":false},\"legendPosition\":\"right\",\"seriesParams\":[{\"data\":{\"id\":\"1\",\"label\":\"Duration in seconds\"},\"drawLinesBetweenPoints\":true,\"mode\":\"stacked\",\"show\":\"true\",\"showCircles\":true,\"type\":\"histogram\",\"valueAxis\":\"ValueAxis-1\"}],\"thresholdLine\":{\"color\":\"#34130C\",\"show\":false,\"style\":\"full\",\"value\":10,\"width\":1},\"times\":[],\"type\":\"histogram\",\"valueAxes\":[{\"id\":\"ValueAxis-1\",\"labels\":{\"filter\":false,\"rotate\":0,\"show\":true,\"truncate\":100},\"name\":\"LeftAxis-1\",\"position\":\"left\",\"scale\":{\"mode\":\"normal\",\"type\":\"linear\"},\"show\":true,\"style\":{},\"title\":{\"text\":\"Duration in seconds\"},\"type\":\"value\"}]},\"title\":\"URL Vs Duration\"}"},"id":"c57f2970-11a6-11ea-b955-77a7a3a8bf45","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzU5LDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"query\":\"httpReqUserAgent : * \",\"language\":\"kuery\"},\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\",\"filter\":[]}"},"title":"Top User Agents","uiStateJSON":"{\"vis\":{\"params\":{\"sort\":{\"columnIndex\":null,\"direction\":null}}}}","version":1,"visState":"{\"type\":\"table\",\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"count\",\"schema\":\"metric\",\"params\":{\"customLabel\":\"Number of Requests\"}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"bucket\",\"params\":{\"field\":\"httpReqUserAgent.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":10,\"otherBucket\":false,\"otherBucketLabel\":\"Other\",\"missingBucket\":false,\"missingBucketLabel\":\"Missing\"}}],\"params\":{\"perPage\":10,\"showPartialRows\":false,\"showMetricsAtAllLevels\":false,\"sort\":{\"columnIndex\":null,\"direction\":null},\"showTotal\":false,\"totalFunc\":\"sum\",\"percentageCol\":\"\"},\"title\":\"Top User Agents\"}"},"id":"dc4905e0-2285-11eb-be28-417e19aaee8d","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzYwLDFd"}
{"attributes":{"description":"","kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"query\":\"transCltRTT : * \",\"language\":\"kuery\"},\"indexRefName\":\"kibanaSavedObjectMeta.searchSourceJSON.index\",\"filter\":[]}"},"title":"URL Vs Client RTT","uiStateJSON":"{}","version":1,"visState":"{\"type\":\"histogram\",\"aggs\":[{\"id\":\"1\",\"enabled\":true,\"type\":\"avg\",\"schema\":\"metric\",\"params\":{\"field\":\"transCltRTT\",\"customLabel\":\"Client Network Delay in ms\"}},{\"id\":\"2\",\"enabled\":true,\"type\":\"terms\",\"schema\":\"segment\",\"params\":{\"field\":\"httpReqUrl.keyword\",\"orderBy\":\"1\",\"order\":\"desc\",\"size\":5,\"otherBucket\":true,\"otherBucketLabel\":\"Other\",\"missingBucket\":true,\"missingBucketLabel\":\"Missing\",\"customLabel\":\"HTTP URL\"}},{\"id\":\"3\",\"enabled\":true,\"type\":\"range\",\"schema\":\"group\",\"params\":{\"field\":\"transCltRTT\",\"ranges\":[{\"from\":0,\"to\":500},{\"from\":500,\"to\":1000},{\"from\":1000,\"to\":1500},{\"from\":1500,\"to\":2000},{\"from\":2000,\"to\":2500},{\"from\":2500,\"to\":3000},{\"from\":3000}],\"customLabel\":\"Client RTT in ms\"}}],\"params\":{\"addLegend\":true,\"addTimeMarker\":false,\"addTooltip\":true,\"categoryAxes\":[{\"id\":\"CategoryAxis-1\",\"labels\":{\"filter\":true,\"rotate\":75,\"show\":true,\"truncate\":100},\"position\":\"bottom\",\"scale\":{\"type\":\"linear\"},\"show\":true,\"style\":{},\"title\":{},\"type\":\"category\"}],\"dimensions\":{\"series\":[{\"accessor\":1,\"aggType\":\"range\",\"format\":{\"id\":\"range\",\"params\":{\"id\":\"number\"}},\"params\":{}}],\"x\":{\"accessor\":0,\"aggType\":\"terms\",\"format\":{\"id\":\"terms\",\"params\":{\"id\":\"string\",\"missingBucketLabel\":\"Missing\",\"otherBucketLabel\":\"Other\"}},\"params\":{}},\"y\":[{\"accessor\":2,\"aggType\":\"count\",\"format\":{\"id\":\"number\"},\"params\":{}}]},\"grid\":{\"categoryLines\":false},\"labels\":{\"show\":false},\"legendPosition\":\"right\",\"seriesParams\":[{\"data\":{\"id\":\"1\",\"label\":\"Client Network Delay in ms\"},\"drawLinesBetweenPoints\":true,\"mode\":\"stacked\",\"show\":\"true\",\"showCircles\":true,\"type\":\"histogram\",\"valueAxis\":\"ValueAxis-1\"}],\"thresholdLine\":{\"color\":\"#34130C\",\"show\":false,\"style\":\"full\",\"value\":10,\"width\":1},\"times\":[],\"type\":\"histogram\",\"valueAxes\":[{\"id\":\"ValueAxis-1\",\"labels\":{\"filter\":false,\"rotate\":0,\"show\":true,\"truncate\":100},\"name\":\"LeftAxis-1\",\"position\":\"left\",\"scale\":{\"mode\":\"normal\",\"type\":\"linear\"},\"show\":true,\"style\":{},\"title\":{\"text\":\"Client Network Delay in ms\"},\"type\":\"value\"}]},\"title\":\"URL Vs Client RTT\"}"},"id":"1270bad0-2283-11eb-be28-417e19aaee8d","migrationVersion":{"visualization":"7.8.0"},"references":[{"id":"cf067c60-10e7-11ea-b955-77a7a3a8bf45","name":"kibanaSavedObjectMeta.searchSourceJSON.index","type":"index-pattern"}],"type":"visualization","updated_at":"2020-11-09T13:05:05.621Z","version":"WzYxLDFd"}
{"attributes":{"description":"","hits":0,"kibanaSavedObjectMeta":{"searchSourceJSON":"{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"},"optionsJSON":"{\"hidePanelTitles\":false,\"useMargins\":true}","panelsJSON":"[{\"embeddableConfig\":{},\"gridData\":{\"h\":6,\"i\":\"026e013e-d34d-4801-99ac-6c02df42758d\",\"w\":12,\"x\":0,\"y\":0},\"panelIndex\":\"026e013e-d34d-4801-99ac-6c02df42758d\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_0\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":6,\"i\":\"498c2a9f-57a5-4462-9813-5b8657e5cc24\",\"w\":35,\"x\":12,\"y\":0},\"panelIndex\":\"498c2a9f-57a5-4462-9813-5b8657e5cc24\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_1\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":12,\"i\":\"27cf0d98-f253-4e40-875d-84a9f6522e30\",\"w\":24,\"x\":0,\"y\":6},\"panelIndex\":\"27cf0d98-f253-4e40-875d-84a9f6522e30\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_2\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":12,\"i\":\"70137540-b809-4615-90fa-867097551760\",\"w\":24,\"x\":24,\"y\":6},\"panelIndex\":\"70137540-b809-4615-90fa-867097551760\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_3\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":19,\"i\":\"2382b3fb-f228-4274-8c5f-b63192ec0053\",\"w\":24,\"x\":0,\"y\":18},\"panelIndex\":\"2382b3fb-f228-4274-8c5f-b63192ec0053\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_4\"},{\"embeddableConfig\":{\"title\":\"URL Vs Duration\",\"vis\":{\"legendOpen\":false}},\"gridData\":{\"h\":19,\"i\":\"ec2d0b37-fea7-4963-b550-9f7a537cf82d\",\"w\":24,\"x\":24,\"y\":18},\"panelIndex\":\"ec2d0b37-fea7-4963-b550-9f7a537cf82d\",\"title\":\"URL Vs Duration\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_5\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":15,\"i\":\"291c9702-9c97-4ebb-9d4d-fee46d995bf7\",\"w\":24,\"x\":0,\"y\":37},\"panelIndex\":\"291c9702-9c97-4ebb-9d4d-fee46d995bf7\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_6\"},{\"embeddableConfig\":{},\"gridData\":{\"h\":15,\"i\":\"4c029db7-3215-4b82-b505-eabd189e1283\",\"w\":24,\"x\":24,\"y\":37},\"panelIndex\":\"4c029db7-3215-4b82-b505-eabd189e1283\",\"version\":\"7.8.0\",\"panelRefName\":\"panel_7\"}]","timeRestore":false,"title":"App Transaction dashboard","version":1},"id":"b7287a90-11a9-11ea-b955-77a7a3a8bf45","migrationVersion":{"dashboard":"7.3.0"},"references":[{"id":"c6e18cb0-11b3-11ea-b955-77a7a3a8bf45","name":"panel_0","type":"visualization"},{"id":"dd7708e0-11b5-11ea-b955-77a7a3a8bf45","name":"panel_1","type":"visualization"},{"id":"2238aa80-11a0-11ea-b955-77a7a3a8bf45","name":"panel_2","type":"visualization"},{"id":"1328ca80-11b3-11ea-b955-77a7a3a8bf45","name":"panel_3","type":"visualization"},{"id":"f2cbd110-119e-11ea-b955-77a7a3a8bf45","name":"panel_4","type":"visualization"},{"id":"c57f2970-11a6-11ea-b955-77a7a3a8bf45","name":"panel_5","type":"visualization"},{"id":"dc4905e0-2285-11eb-be28-417e19aaee8d","name":"panel_6","type":"visualization"},{"id":"1270bad0-2283-11eb-be28-417e19aaee8d","name":"panel_7","type":"visualization"}],"type":"dashboard","updated_at":"2020-11-09T13:05:05.621Z","version":"WzYyLDFd"}
```
