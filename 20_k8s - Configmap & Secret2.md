

# ConfigMaps and Secrets- Part2

하기의 기술된 내용은 [Google Cloud의 Kubernetes ConfigMaps and Secrets](https://medium.com/google-cloud/kubernetes-configmaps-and-secrets-part-2-3dc37111f0dc)의 내용은 기반으로 작성된 내용입니다. 

때때로 애플리케이션에서 참조변수로 사용되는 데이터가 관리가 필요할만큼 여러개가 셋팅되어지는 경우가 있습니다. 기존 VM기반의 클러스터환경에서도 마찬가지였지만, 이러한 경우에는 주로 데이터를 파일로 그룹화하여 애플리케이션이 기동할때 설정파일을 읽도록 하곤 합니다.

Kubernetes에서도 마찬가지로 이러한 이유로 Kubernetes의 ConfigMap과 Secret객체를 Pod에서 디렉터리 또는 파일로 마운트 할 수 있습니다. 환경변수와는 달리 이렇게 마운트된 ConfigMap과 Secret 객체는 파일이 변경되면 새파일을 다시 마운트하거나 쓸필요없이 실행중인 Pod에 Push되어 반영되게 됩니다. 따라서 환경변수에 비해 훨씬더 강력한 효과를 가져올 수 있습니다. 또한 여러 개의 파일을 하나의 ConfigMap 또는 Secret Object에 매핑하고 한번에 모두 디렉터리로 마운트 할 수 있습니다. 



### Read from a file

여기 아래와 같이 file.js라는 소스코드 파일이 있습니다. file.js는 앞서 Part1과는 다르게 소스에서 설정파일을 읽어들여 참조변수를 설정합니다.

우리가 이번 Part에서 하고자 하는 것은 위의 소스코드 중 설정파일을 읽는 부분에서 docker와 kubernetes에서 설정 할 수 있는 방법에 대해 실습해 보는 것입니다.

먼저 아래와 같이 설정데이터가 담겨 있는 config.json, secret.json 파일을 생성하겠습니다.

```
mkdir -p /lab/config2
cd /lab/config2
echo '{"LANGUAGE":"English"}' > /lab/config2/config.json
echo '{"API_KEY":"123456789"}' > /lab/config2/secret.json
```

```bash
gedit /lab/config2/file.js
```

```js
// Copyright 2017, Google, Inc.
// Licensed under the Apache License, Version 2.0 (the "License")
var http = require('http');
var fs = require('fs');
var server = http.createServer(function (request, response) {
  fs.readFile('/lab/config2/config.json', function (err, config) {
    if (err) return console.log(err);
    const language = JSON.parse(config).LANGUAGE;
    fs.readFile('/lab/config2/secret.json', function (err, secret) {
      if (err) return console.log(err);
      const API_KEY = JSON.parse(secret).API_KEY;
      response.write(`Language: ${language}\n`);
      response.write(`API Key: ${API_KEY}\n`);
      response.end(`\n`);
    });
  });
});
server.listen(3000);
```

정상적으로 실행 되었는지 확인하기 위해 위에서 생성한 file.js를 node로 실행해보겠습니다.

```
cd /lab/config2
node file.js
```

정상적으로 실행되었다면 브라우저에서 localhost:3000을 호출시 아래와 같이 표시됩니다.

```
Language: English
API Key: 123456789
```





### Mount the files using Docker volumes

첫번째 단계는 Docker Volume을 활용하여 설정파일을 로딩하는 방법을 실습해보겠습니다.

사실 이번 실습은 Docker Volume에 대한 내용을 알고 있다면 너무나도 쉬운 예제입니다. Docker 실행시 Host의 마운트 경로와 Container의 마운트 경로를 지정하면 되기 때문입니다.

```
mkdir -p /lab/config2/
cd /lab/config2/
gedit /lab/config2/Dockerfile
```

```
# Copyright 2017, Google, Inc.
# Licensed under the Apache License, Version 2.0 (the "License")
FROM node:8
EXPOSE 3000
VOLUME /lab/config2/
COPY . /lab/config2/
WORKDIR /lab/config2/
CMD ["node", "file.js"]
```

 dockerfile을 빌드합니다.

```
docker build -t localhost:5000/envtest:v2 .
```

빌드한 이미지를 실행합니다.

```
docker run -d -p 3000:3000 -v /lab/config2/:/lab/config2/ localhost:5000/envtest:v2
```

브라우저에서 localhost:3000을 호출하면 아래와 같이 환경변수로 등록한 참조데이터들이 브라우저에 표시되는 것을 확인할 수 있습니다.

```
Language: English
API Key: 123456789
```

당연한 이야기이지만, 아래와 같이 Host의 설정파일을 변경하면 Container안의 마운트된 경로의 설정파일도 변경이 되어, 브라우저에서 다시 애플리케이션을 호출하면 변경된 내용으로 바뀌는 것을 확인 할 수 있습니다.

```
echo '{"LANGUAGE":"Spanish"}' > /lab/config2/config.json
```



### Creating the ConfigMap and Secret

이번에는 설정파일로부터 ConfigMap, Secret객체를 생성하는 과정입니다.

설정파일로부터 Secret객체를 생성합니다.

```
kubectl create secret generic my-secret --from-file=/lab/config2/secret.json
```

설정파일로부터 ConfigMap객체를 생성합니다.

```
kubectl create configmap my-config --from-file=/lab/config2/config.json
```

배포한 Object를 확인해봅시다.

```
# kubectl get secret
NAME                  TYPE                                  DATA   AGE
apikey                Opaque                                1      7s

# kubectl get configmap
NAME       DATA   AGE
language   1      6s

```



### Using ConfigMaps and Secrets as files

마지막으로 위에서 생성한 ConfigMap과 Secret객체를 Pod의 Spec으로 마운트하는 내용입니다. 중요한 것은 앞서 Part1과는 다르게 Pod의 환경변수로 명세한 것이 아니라 container spec의 volumeMount 속성을 사용했다는 것입니다. 이러한 설정은 위의 Docker의 Volume과 마찬가지로 컨테이너가 시작될 때 자동으로 마운트됩니다.

```
gedit /lab/config2/file.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: envtest
  labels:
    name: envtest
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    protocol: TCP
  selector:
    name: envtest
---
# Copyright 2017, Google, Inc.
# Licensed under the Apache License, Version 2.0 (the "License")
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: envtest
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: envtest
    spec:
      containers:
      - name: envtest
        image: localhost:5000/envtest:v2
        ports:
        - containerPort: 3000
        volumeMounts:
          - name: my-config
            mountPath: /lab/config2
          - name: my-secret
            mountPath: /lab/config2
      volumes:
      - name: my-config
        configMap:
          name: my-config
      - name: my-secret
        secret:
          secretName: my-secret
```



### Updating with no downtime!

Part1의 환경변수설정과는 달리, Volume은 Container가 실행중인 상태에서 동적으로 다시 마운트가 가능합니다. 즉, 실행중인 Container를 재시작하지 않아도 변경된 ConfigMap과 Secret을 사용할 수 있다는 의미입니다.

예로써, language를 Klingon 으로 변경하고 클러스터에 반영하면

```
echo '{"LANGUAGE":"Klingon"}' > /lab/config2/config.json
kubectl create configmap my-config \
  --from-file=/lab/config2/config.json \
  -o yaml --dry-run | kubectl replace -f -
```

몇 초 후에 새로운 파일이 자동으로 Container안으로 푸쉬됩니다.

반대로 이러한 자동 Update가 여러개의 Pod에 발생하게 되면 일부 컨테이너가 다른 컨테이너에 비해 먼저 업데이트 된 설정을 가져와서 불일치가 발생할 수 있습니다. 이런 문제점이 문제가 된다면 새로운 ConfigMap또는 Secret을 만들고 Deployment를 업데이트하거나 새로 만드는 것이 좋습니다.

