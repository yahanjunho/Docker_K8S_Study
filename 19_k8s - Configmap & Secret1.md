

# ConfigMaps and Secrets - Part1

하기의 기술된 내용은 [Google Cloud의 Kubernetes ConfigMaps and Secrets](https://medium.com/google-cloud/kubernetes-configmaps-and-secrets-68d061f7ab5b)의 내용은 기반으로 작성된 내용입니다. 

모든 애플리케이션은 참조되는 데이터 혹은 값을 가질 수 있습니다. 예를 들어 API Key, 토큰, 어떠한 계정의 비밀번호 등입니다. 

이러한 참조데이터들은 만약 애플리케이션의 규모가 굉장히 작은 규모라면 애플리케이션 소스코드에 하드코딩으로 설정하는 것도 나쁘지 않습니다. 그러나 애플리케이션의 규모가 확장성 있고, 크다면 하드코딩된 참조데이터들은 개발자나 운영자에 의한 실수로 심각한 오류상황을 야기 시킬 수 있기 때문에 주로 애플리케이션을 구성하기 위한 셋팅파일이나 OS 환경변수로 저장하여 애플리케이션에서 참조되어집니다.

이러한 설정은 소스코드에 하드코딩을 피하게 해주고, 참조데이터들의 중앙집중화된 관리가 용이하게 해주었습니다. 예로써 애플리케이션에서 사용되는 설정변수를 변경하기 위해서 단순히 환경변수나 설정파일의 값을 바꾸는 것만으로 반영이 되었었습니다. 이러한 방식은 썩 괞잖은 방법입니다.

문제는 컨테이너 환경에서는 OS환경변수나 설정파일로 참조데이터를 셋팅할 때 문제가 발생할 수 있다는데 있습니다. 예를 들어 OS환경변수에 참조변수를 설정한다는 것은 컨테이너 환경에서는 Dockerfile에 변수로 지정한다는 의미입니다. 그런데 만약 두 개의 다른 컨테이너가 동일한 데이터를 참조한다면 어떻게해야할까요? Dockerfile이 아니라 기존방식대로 OS환경변수로 관리한다면, 멀티호스팅 상태에서 어떻게 여러대의 컴퓨터에 동일한 데이터를 설정할 수 있을까요?

우리는 여기서 단계별로 환경변수를 설정하는 방법에대해 실습해볼 예정입니다. 그 단계라는 것은 위에서 언급한 하드코딩 -> OS 환경변수 -> Dockerfile -> Kubernetes ConfigMap 이며, 이러한 과정을 통해 Kubernetes ConfigMap과 Secret을 사용하고 관리하는 방법에대해 이해 할 수 있게 됩니다.



### The Hardcoded App

여기 아래와 같이 hardcode.js라는 소스코드 파일이 있다고 가정하겠습니다.

우리가 하고자 하는 것은 위의 소스코드 중 Language와 API key를 단계별로 변경하고자 하는 것입니다.

```
mkdir -p /lab/config1/
cd /lab/config1/
gedit /lab/config1/hardcode.js
```

```js
// Copyright 2017, Google, Inc.
// Licensed under the Apache License, Version 2.0 (the "License")
var http = require('http');
var server = http.createServer(function (request, response) {
  const language = 'English';
  const API_KEY = '123456789';
  response.write(`Language: ${language}\n`);
  response.write(`API Key: ${API_KEY}\n`);
  response.end(`\n`);
});
server.listen(3000);
```

위 소스를 실행하기 위해서 먼저 아래와 같이 node.js를 설치하겠습니다.

```bash
apt-get update

curl -sL https://deb.nodesource.com/setup_5.x | sudo -E bash -
apt-get install -y nodejs
apt-get install -y build-essential
apt-get install -y npm
```

설치가 정상적으로 되었는지 확인하기 위해 위에서 생성한 hardcode.js를 node로 실행해보겠습니다.

```
cd /lab/config1
node hardcode.js
```

정상적으로 실행되었다면 브라우저에서 localhost:3000을 호출시 아래와 같이 표시됩니다.

```
Language: English
API Key: 123456789
```



### Step 1: Using Environment Variables

OS 환경변수를 활용하여 참조데이터를 설정하는 방법에 대해 실습해보겠습니다.

```bash
mkdir -p /lab/config1
gedit /lab/config1/envvars.js
```

```js
// Copyright 2017, Google, Inc.
// Licensed under the Apache License, Version 2.0 (the "License")
var http = require('http');
var server = http.createServer(function (request, response) {
  const language = process.env.LANGUAGE;
  const API_KEY = process.env.API_KEY;
  response.write(`Language: ${language}\n`);
  response.write(`API Key: ${API_KEY}\n`);
  response.end(`\n`);
});
server.listen(3000);
```

위의 코드에서 보듯이 환경변수로 참조데이터를 설정하는 것은 소스코드상의 하드코딩을 없애줍니다.

이제 OS 환경변수로 LANGUAGE, API_KEY를 설정해보겠습니다.

```
export LANGUAGE=English
export API_KEY=123456789
```

envvars.js 실행해 결과를 확인해봅시다.

```
cd /lab/config1
node envvars.js
```

브라우저에서 localhost:3000을 호출하면 아래와 같이 환경변수로 등록한 참조데이터들이 브라우저에 표시되는 것을 확인할 수 있습니다.

```
Language: English
API Key: 123456789
```



### Step 2: Moving to Docker Environment Variables

When you move to a containerized solution, you can no longer depend on the host’s environment variables. Each container has its own environment, so it is important to make sure the container’s environment is properly configured. Thankfully, Docker makes it easy to build containers with environment variables baked in. In your Dockerfile, you can specify them with the ENV directive.

```
mkdir -p /lab/config1/
cd /lab/config1/
gedit /lab/config1/Dockerfile
```

```
# Copyright 2017, Google, Inc.
# Licensed under the Apache License, Version 2.0 (the "License")
FROM node:8
EXPOSE 3000
ENV LANGUAGE English
ENV API_KEY 123456789
RUN mkdir -p /lab/config1/
COPY . /lab/config1/
WORKDIR /lab/config1/
CMD ["node", "envvars.js"]
```

 dockerfile을 빌드합니다.

```
docker build -t localhost:5000/envtest:v1 .
```

빌드한 이미지를 실행합니다.

```
docker run -d -p 3000:3000 localhost:5000/envtest:v1
```

브라우저에서 localhost:3000을 호출하면 아래와 같이 환경변수로 등록한 참조데이터들이 브라우저에 표시되는 것을 확인할 수 있습니다.

```
Language: English
API Key: 123456789
```



### Step 3: Moving to Kubernetes Environment Variables

When moving from Docker to Kubernetes, things change once again. You might use the same Docker container in multiple kubernetes deployments or you might want to do A/B testing with your deployments where you use the same container with different configurations.

Just like the Dockerfile, you can specify environment variables directly in your Kubernetes Deployment YAML file. This means each deployment can get a customized environment.

```bash
mkdir -p /lab/config1/
cd /lab/config1/
gedit /lab/config1/embededenv.yaml
```

```yaml
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
        image: localhost:5000/envtest:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        env:
        - name: LANGUAGE
          value: "English"
        - name: API_KEY
          value: "123456789"
```

kubectl apply 명령어로 위에서 작성된 yaml파일을 클러스터에 배포해봅시다.

```
kubectl apply -f /lab/config1/embededenv.yaml
```

배포후 vm으로 노출된 포트를 조회하여 브라우저에서 해당 url을 호출해보면 위에서와 마찬가지로 우리가 원하는 결과가 표시됩니다.

```
# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
envtest      NodePort    172.168.1.191   <none>        3000:31256/TCP   14s
```

```
Language: English
API Key: 123456789
```



### Step 4: Centralized Configuration with Kubernetes Secrets and ConfigMaps

위의 과정중 Step2, 3에서 실습한 dockerfile, kubernetes pod spec 에 환경변수로 설정하는 것은 무슨 문제가 있을까요? 무엇보다 dockerfile 또는 pod spec에 종속되어 있다는 점입니다. 만약 환경변수를 변경하려면 docker빌드를 새로하거나 pod spec을 변경해 다시 배포해야합니다.

이러한 문제점때문에 kubernetes는 ConfigMap 과 Secret이라는 Object로 이 문제를 해결합니다. 기본적으로 이 두가지 Object는 Key-Value방식의 Object라는 점은 같지만, 차이점은 Secret은 Base64 인코딩으로 암호화가 된다는 점입니다. 그렇기 때문에 비공개나 보안을 요구하는 참조데이터인 경우에는 Secret을 사용하고 그렇치 않은 경우는 ConfigMap을 사용합니다.

아래와 같이 API key는 Secret , language는 ConfigMap Object로 생성해 위에서 배포한 deployment를 다시 배포해봅시다.

```
kubectl create secret generic apikey --from-literal=API_KEY=123456789
```

```
kubectl create configmap language --from-literal=LANGUAGE=English
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

ConfigMap과 Secret을 생성했다면, 위에서 배포한 deployment에 ConfigMap과 Secret객체의 키값으로 연결 할 수 있습니다.

```
cd /lab/config1/
gedit /lab/config1/final.yaml
```

```yaml
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
        image: localhost:5000/envtest:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        env:
        - name: LANGUAGE
          valueFrom:
            configMapKeyRef:
              name: language
              key: LANGUAGE
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: apikey
              key: API_KEY
```

```
kubectl apply -f /lab/config1/final.yaml
```



### Updating Secrets and ConfigMaps

Kubernetes에서 ConfigMap과 Secret을 통해 환경변수를 관리한다는 것은 변수의 값을 변경하려는 경우 코드를 변경하거나 컨테이너를 다시 빌드할 필요가 없다는 것을 의미합니다.

Pod가 시작될때 환경변수를 Cache해 셋팅하므로, 만약 위에서 설정한 환경변수를 변경하기 위해서는 ConfigMap과 Secret을 변경하고 Pod를 재시작하면됩니다.

먼저 위에서 생성한 ConfigMap과 Secret을 변경합니다.

```
kubectl create configmap language --from-literal=LANGUAGE=Spanish \
        -o yaml --dry-run | kubectl replace -f -
kubectl create secret generic apikey --from-literal=API_KEY=098765 \
        -o yaml --dry-run | kubectl replace -f -
```

Pod를 재시작하는 방법에는 여러가지가 있지만, 여기서는 간단히 테스트하기위해 강제로 Pod를 삭제하도록하겠습니다.

```
kubectl delete pod -l name=envtest
```
