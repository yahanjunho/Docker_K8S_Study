# The Chart Template Developer's Guide

# Getting Started with a Chart Template

## A Starter Chart

```
# mkdir /lab/helm/chart
# cd /lab/helm/chart

# helm create mychart
Creating mychart

# tree
.
└── mychart
    ├── charts
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   ├── ingress.yaml
    │   ├── NOTES.txt
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

간단한 예제를 만들어 보기 위해 템플릿으로 생성된 파일들을 삭제합니다.

```
# rm -rf mychart/templates/*
```



## A First Template

간단한 Kubernetes configmap Object를 하나 생성해 배포해보도록하겠습니다.

```
# gedit /lab/helm/chart/mychart/templates/configmap.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

helm install로 클러스터에 배포합니다.

```
# helm install ./mychart
NAME:   alliterating-eel
LAST DEPLOYED: Wed Mar 20 13:15:12 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME               DATA  AGE
mychart-configmap  1     <invalid>


root@k8sm:/lab/helm/chart# helm ls
NAME                	REVISION	UPDATED                 	STATUS  	CHART            	APP VERSION	NAMESPACE
alliterating-eel    	1       	Wed Mar 20 13:15:12 2019	DEPLOYED	mychart-0.1.0    	1.0        	default  
excited-umbrellabird	1       	Wed Mar 20 12:02:31 2019	DEPLOYED	chartmuseum-2.1.0	0.8.2      	default  
mortal-termite      	1       	Wed Mar 20 12:54:49 2019	DEPLOYED	wordpress-5.7.0  	5.1.1      	default 
```

배포된 실제 차트를 확인하기 위해서는 get명령어로 manifest를 조회하면 됩니다.

```
# helm get manifest alliterating-eel

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"

```

만약 이상태에서 한번대 동일한 헬름차트를 배포하면, 이미 생성된 Kubernetes configmap Object가 있기 때문에 에러가 발생합니다.

```
# helm install ./mychart
Error: release terrific-elephant failed: configmaps "mychart-configmap" already exists
# helm ls
NAME                	REVISION	UPDATED                 	STATUS  	CHART            	APP VERSION	NAMESPACE
alliterating-eel    	1       	Wed Mar 20 13:15:12 2019	DEPLOYED	mychart-0.1.0    	1.0        	default  
excited-umbrellabird	1       	Wed Mar 20 12:02:31 2019	DEPLOYED	chartmuseum-2.1.0	0.8.2      	default  
mortal-termite      	1       	Wed Mar 20 12:54:49 2019	DEPLOYED	wordpress-5.7.0  	5.1.1      	default  
terrific-elephant   	1       	Wed Mar 20 13:18:32 2019	FAILED  	mychart-0.1.0    	1.0        	default
```

생성한 릴리즈를 `helm delet`명령어로 삭제합니다.

```
# helm delete alliterating-eel
# helm delete terrific-elephant
```



### Adding a Simple Template Call

위의 예제에서 하드코딩된 name값을 헬름 내장 객체 release name으로 변경해보겠습니다.

**TIP:**  `name:` 필드는 DNS의 제한때문에 63 character로 제한됩니다. 이에 대응해 release name은 53 character로 제한됩니다.

`configmap.yaml`을 아래와 같이 수정한 후 다시 install을 진행하겠습니다.

```
# gedit /lab/helm/chart/mychart/templates/configmap.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

```
# helm install ./mychart
NAME:   good-abalone
LAST DEPLOYED: Wed Mar 20 14:02:57 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                    DATA  AGE
good-abalone-configmap  1     <invalid>
```

Helm에서는 위와 같이 편집중인 차트를 미리 확인 하기 위한 모드를 제공하고 있습니다. install명령어에 아래와 같이 옵션을 주면 실제로 install은 되지 않고 템플릿이 어떻게 랜더링되는지 확인 할 수 있습니다.

```
# helm install --debug --dry-run ./mychart
[debug] Created tunnel using local port: '33941'

[debug] SERVER: "127.0.0.1:33941"

[debug] Original chart version: ""
[debug] CHART PATH: /lab/helm/chart/mychart

NAME:   invisible-abalone
REVISION: 1
RELEASED: Wed Mar 20 14:12:30 2019
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
  tag: stable
ingress:
  annotations: {}
  enabled: false
  hosts:
  - host: chart-example.local
    paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []

HOOKS:
MANIFEST:

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: invisible-abalone-configmap
data:
  myvalue: "Hello World"
```

# Values Files

이번에는 Values 파일을 활용하여 템플릿을 변경해보도록 하겠습니다.

참고로 Values File은 아래와 같은 특징을 가지고 있습니다.

- 차트안에 `values.yaml` 파일명으로 생성된 파일입니다.
- 상위 차트의`values.yaml`에 의해 재정의 될 수 있습니다.
- `helm install` or `helm upgrade` 시 -f 옵션으로 변경될 수 있습니다.(`helm install -f myvals.yaml ./mychart`)
- 개별적인 속성 값은 `--set`옵션으로 변경할 수 있습니다. (such as `helm install --set foo=bar ./mychart`)

위의 예제에서 `mychart/values.yaml`을 변경해 ConfigMap template에 반영해봅시다.

```
# gedit /lab/helm/chart/mychart/values.yaml
```

```
favoriteDrink: coffee
```

그리고 template 폴더안의 configmap.yaml파일에 해당내용을 반영합니다.

```
# gedit /lab/helm/chart/mychart/templates/configmap.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

렌데링해서 결과를 확인해 봅니다.

```
# helm install --dry-run --debug ./mychart
[debug] Created tunnel using local port: '46425'

[debug] SERVER: "127.0.0.1:46425"

[debug] Original chart version: ""
[debug] CHART PATH: /lab/helm/chart/mychart

NAME:   kneeling-puffin
REVISION: 1
RELEASED: Wed Mar 20 15:35:21 2019
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
favoriteDrink: coffee

HOOKS:
MANIFEST:

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kneeling-puffin-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

 `values.yaml`파일에 정의한 `coffee`가 템플릿에 반영된 것을 확인 할 수 있습니다.

이렇게 Values로 정의 된 속성은 위에서 언급했지만 install 명령어시 --set 옵션으로 아래와 같이 오버라이딩 될 수 있습니다.

```
# helm install --dry-run --debug --set favoriteDrink=slurm ./mychart
[debug] Created tunnel using local port: '36735'

[debug] SERVER: "127.0.0.1:36735"

[debug] Original chart version: ""
[debug] CHART PATH: /lab/helm/chart/mychart

NAME:   dusty-nightingale
REVISION: 1
RELEASED: Wed Mar 20 15:37:58 2019
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
favoriteDrink: slurm

COMPUTED VALUES:
favoriteDrink: slurm

HOOKS:
MANIFEST:

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dusty-nightingale-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

Values 파일은 아래와 같이 구조적으로 속성을 정의하고 사용할 수도 있습니다.

```
favorite:
  drink: coffee
  food: pizza
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

# Template Functions and Pipelines

이번에는 Template 파일들 안에서 사용할 수 있는 함수와 Pipeline에 대해 살펴보겠습니다.

ConfigMap  객체에 문자열을 삽입할 때 quote 함수를 사용할 수 있습니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

Template 함수들은 `functionName arg1 arg2...`. 와 같은 신텍스를 가집니다.

Helm은 60개 이상의 함수들을 사용할 수 있습니다. 해당내용은  [Go template language](https://godoc.org/text/template) , [Sprig template library](https://godoc.org/github.com/Masterminds/sprig)에서 확인 할 수 있습니다.



## Pipelines

UNIX의 개념과 동일

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

위의 예제는 아래와 같이 렌더링됩니다.

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```



## Using the `default` function

`default`함수는 비교적 자주사용되는 함수로 Value값이 생략 된 경우 템플릿 내부에 기본값을 지정할 수 있게합니다.

```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

`values.yaml`파일의 drink 속성을 주석으로 처리하면

```
favorite:
  #drink: coffee
  food: pizza
```

`helm install --dry-run --debug ./mychart` 시 아래와 같이 랜더링 됩니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

실제 차트에서는 모든 정적값은 values.yaml 파일에 존재해야 합니다. 따라서 `default`함수는 아래와 같이 활용되는 것이 더 적합합니다.

```
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```



## Operators are functions

아래와 같은 연산자 함수를 사용할 수 있습니다.

 `eq`, `ne`, `lt`, `gt`, `and`, `or`, `not` 

```
{{/* .Values.fooString이 존재하고, .Values.fooString이 "foo"와 동일할때 */}}
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}

```

# Flow Control

Flow Control로 템플릿 구조의 흐름을 제어할 수 있습니다.

- `if`/`else` 조건 블록 생성
- `with`범위 지정
- `range`,  "for each"-style loop

In addition to these, it provides a few actions for declaring and using named template segments:

- `define` declares a new named template inside of your template
- `template` imports a named template
- `block` declares a special kind of fillable template area



## If/Else

```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

파이프라인은 value가 아래와 같을 때 false로 인식합니다.

- a boolean false
- a numeric zero
- an empty string
- a `nil` (empty or null)
- an empty collection (`map`, `slice`, `tuple`, `dict`, `array`)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if and .Values.favorite.drink (eq .Values.favorite.drink "coffee") }}mug: true{{ end }}
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```



## Modifying scope using `with`

with는 범위를 제어할때 사용됩니다.

```
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```



## Looping with the `range` action

 `foreach` loop와 동일한 성격의 제어문으로 예제를 아래와 같이 수정하겠습니다.

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```



```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```



```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

# Variables

다른 프로그램언어와 마찬가지로 변수를 생성해 활용할 수 있습니다.

위의 예제를 아래와 같이 수정하면 에러가 발생하는데 원인은 .Release.Name은 with블럭 안에서는 사용될 수 없기 때문입니다. 이런 경우에 변수를 생성하여 활용할 수 있습니다.

```
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```



```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```



```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: viable-badger-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: viable-badger
```

이러한 변수를 range 문과 같이 사용하면 효과적일 수 있습니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end}}
```



```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eager-rabbit-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

# Named Templates

이번에는 Named Templates을 만들어 다른 템플릿에 사용하는 방법을 알아보겠습니다. 일반적으로 Named Templates은  템플릿에 사용할 때는  `define`, `template`, `block` ,`include` 함수를 사용하는데 자주사용되는 것은 `_helpers.tpl`파일에 Named Templates을 정의하고 템플릿에 렌더링하는 방식이 많이 사용됩니다.



## Declaring and using templates with `define` and `template`

define을 사용하여 이전예제를 아래와 같이 변경해 봅시다.

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

렌더링한 결과는 아래와 같습니다.

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

`_helpers.tpl`에 템플릿을 생성하여 적용해봅시다.

```
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```



```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

주의해야 할 것은 위에서 보앗듯이 템플릿 이름은 전역으로 사용되므로 두 개의 템플릿이 같은 이름으로 선언되면 마지막에 선언된 템플릿이 사용되는 것에 주의해야합니다.



## The `include` function

간단하게 아래와 같이 템플릿을 정의한경우

```
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}
```

아래와 같이 template을 사용하여 랜더링하면

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" .}}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

원하지 않는 결과가 랜더링됩니다.

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
app_version: "0.1.0+1478129847"
```



이런 경우때문에 Helm에서는 템플레이트의 내용을 파이프라인으로 전달할 수 있는 include를 제공합니다. 이것은  `nindent` 함수와 같이 쓰여 위의 문제를 해결할 수 있습니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{- include "mychart.app" . | nindent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
  {{- include "mychart.app" . | nindent 2 }}
```



```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```



# The .helmignore file

`.helmignore` 파일은 gitignore와 dockeriignore파일과 마찬가지로 헬름차트안에 포함되지 않을 파일들을 명세하는 파일입니다.

```
# comment
.git
*/temp*
*/*/temp*
temp?
```

# Creating a NOTES.txt File

In this section we are going to look at Helm's tool for providing instructions to your chart users. At the end of a `chart install` or `chart upgrade`, Helm can print out a block of helpful information for users. This information is highly customizable using templates.

To add installation notes to your chart, simply create a `templates/NOTES.txt` file. This file is plain text, but it is processed like as a template, and has all the normal template functions and objects available.

Let's create a simple `NOTES.txt` file:

```
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get {{ .Release.Name }}
```

Now if we run `helm install ./mychart` we will see this message at the bottom:

```
RESOURCES:
==> v1/Secret
NAME                   TYPE      DATA      AGE
rude-cardinal-secret   Opaque    1         0s

==> v1/ConfigMap
NAME                      DATA      AGE
rude-cardinal-configmap   3         0s


NOTES:
Thank you for installing mychart.

Your release is named rude-cardinal.

To learn more about the release, try:

  $ helm status rude-cardinal
  $ helm get rude-cardinal
```

Using `NOTES.txt` this way is a great way to give your users detailed information about how to use their newly installed chart. Creating a `NOTES.txt` file is strongly recommended, though it is not required.

# Debugging Templates

Debugging templates can be tricky simply because the templates are rendered on the Tiller server, not the Helm client. And then the rendered templates are sent to the Kubernetes API server, which may reject the YAML files for reasons other than formatting.

There are a few commands that can help you debug.

- `helm lint` is your go-to tool for verifying that your chart follows best practices
- `helm install --dry-run --debug`: We've seen this trick already. It's a great way to have the server render your templates, then return the resulting manifest file.
- `helm get manifest`: This is a good way to see what templates are installed on the server.

When your YAML is failing to parse, but you want to see what is generated, one easy way to retrieve the YAML is to comment out the problem section in the template, and then re-run `helm install --dry-run --debug`:

```
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

The above will be rendered and returned with the comments intact:

```
apiVersion: v1
# some: problem section
#  "bar"
```

This provides a quick way of viewing the generated content without YAML parse errors blocking.



