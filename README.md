# NGINX proxy configuration in OpenShift

## create ns
```
TNS=my-nginx-tests
oc new-project ${TNS}
```

## create configs
```
oc create cm -n ${TNS} nginx-config-proxy --from-file=nginx.conf=./nginx-proxy.conf
oc create cm -n ${TNS} nginx-proxy-index-file --from-file=index.html=./index-proxy.html

oc create cm -n ${TNS} nginx-config-target --from-file=nginx.conf=./nginx-target.conf
oc create cm -n ${TNS} nginx-target-index-file --from-file=index.html=./index-target.html
```

## service account to accomodate nginx standard image
```
cat << EOF | oc create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-anyuid
  namespace: ${TNS}
EOF

oc adm policy add-scc-to-user anyuid -z nginx-anyuid -n ${TNS}
```

## standard image
```
export NGINX_IMAGE=nginx
```

## nginx target deployment 
```
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
```

## nginx proxy deployment 
```
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
```

## routes 
```
oc expose svc nginx
oc expose svc nginx-proxy
```

## tests
```
URL="http://"$(oc get route nginx-proxy -n ${TNS} -o jsonpath='{.spec.host}')

# response from proxy server
curl ${URL} && echo

# response from target server (proxyed)
curl ${URL}/remote && echo

# response from BAMOE DM application (proxyed) 
curl -s -X 'POST' \
  ${URL}/dm/PerformanceLotsOfRules \
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

curl -s -X 'POST' \
  ${URL}/dm/PerformanceAdult \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "age": 20
}' | jq .

curl -s -X 'POST' \
  ${URL}/dm/PerformanceConsumeCpuForTime \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "milliseconds": 100
}' | jq .

time curl -s -H 'accept: application/json' -H 'Content-Type: application/json' -X POST ${URL}/dm/FlightRebooking -d @/tmp/payload | jq .

time curl -s -H 'accept: application/json' -H 'Content-Type: application/json' -X POST ${URL}/dm/FlightRebooking -d @/tmp/payload-big | jq .


```


### update config and restart pods
```
oc get cm | grep nginx | awk '{print $1}' | xargs oc delete cm

oc create cm -n ${TNS} nginx-config-proxy --from-file=nginx.conf=./nginx-proxy.conf
oc create cm -n ${TNS} nginx-proxy-index-file --from-file=index.html=./index-proxy.html

oc create cm -n ${TNS} nginx-config-target --from-file=nginx.conf=./nginx-target.conf
oc create cm -n ${TNS} nginx-target-index-file --from-file=index.html=./index-target.html

oc scale deployment nginx --replicas=0
oc scale deployment nginx-proxy --replicas=0
oc scale deployment nginx --replicas=1
oc scale deployment nginx-proxy --replicas=1
```

Useful refs.

https://dev.to/danielkun/nginx-everything-about-proxypass-2ona