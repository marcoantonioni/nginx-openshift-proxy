TNS=my-nginx-tests
oc new-project ${TNS}

oc create cm -n ${TNS} nginx-config-proxy --from-file=nginx.conf=./nginx-proxy.conf
oc create cm -n ${TNS} nginx-proxy-index-file --from-file=index.html=./index-proxy.html

oc create cm -n ${TNS} nginx-config-target --from-file=nginx.conf=./nginx-target.conf
oc create cm -n ${TNS} nginx-target-index-file --from-file=index.html=./index-target.html


cat << EOF | oc create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-anyuid
  namespace: ${TNS}
EOF

oc adm policy add-scc-to-user anyuid -z nginx-anyuid -n ${TNS}

# usa immagine standard
NGINX_IMAGE=nginx


#--------------------------------------------
# NGINX
cat << EOF | oc apply --force -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: ${TNS}
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccount: nginx-anyuid
      volumes:
        - name: target-config
          configMap:
            name: nginx-config-target
        - name: index-file
          configMap:
            name: nginx-target-index-file
      containers:
        - name: nginx
          image: >-
            ${NGINX_IMAGE}
          ports:
            - containerPort: 8080
          volumeMounts:
            - mountPath: /etc/nginx/
              readOnly: true
              name: target-config
            - mountPath: /opt/app-root/src/
              readOnly: true
              name: index-file
EOF

oc expose deployment nginx


#--------------------------------------------
# PROXY
cat << EOF | oc apply --force -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
  namespace: ${TNS}
spec:
  selector:
    matchLabels:
      app: nginx-proxy
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-proxy
    spec:
      serviceAccount: nginx-anyuid
      volumes:
        - name: proxy-config
          configMap:
            name: nginx-config-proxy
        - name: index-file
          configMap:
            name: nginx-proxy-index-file
      containers:
        - name: nginx-proxy
          image: >-
            ${NGINX_IMAGE}
          ports:
            - containerPort: 8080
          volumeMounts:
            - mountPath: /etc/nginx/
              readOnly: true
              name: proxy-config
            - mountPath: /opt/app-root/src/
              readOnly: true
              name: index-file
EOF

oc expose deployment nginx-proxy

oc expose svc nginx
oc expose svc nginx-proxy


URL=http://$(oc get route nginx-proxy -o jsonpath='{.spec.host}')

curl ${URL} && echo

curl ${URL}/remote && echo

curl ${URL}/PerformanceLotsOfRules && echo

curl -X 'POST' \
  ${URL}/PerformanceLotsOfRules \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "Lev1Input1": true,
  "Lev1Input2": 5,
  "Lev1Input3": true,
  "Lev2Input1": "GOOD",
  "Lev2Input2": true,
  "Lev2Input3": 20,
  "Lev2Input4": "HIGH"
}' | jq .


#------------------------------------------------

oc get cm | grep nginx | awk '{print $1}' | xargs oc delete cm

oc create cm -n ${TNS} nginx-config-proxy --from-file=nginx.conf=./nginx-proxy.conf
oc create cm -n ${TNS} nginx-proxy-index-file --from-file=index.html=./index-proxy.html

oc create cm -n ${TNS} nginx-config-target --from-file=nginx.conf=./nginx-target.conf
oc create cm -n ${TNS} nginx-target-index-file --from-file=index.html=./index-target.html

oc scale deployment nginx --replicas=0
oc scale deployment nginx-proxy --replicas=0
oc scale deployment nginx --replicas=1
oc scale deployment nginx-proxy --replicas=1
