하기의 내용은 개발자포탈의 [Helm : Intro to Chart ](http://70.121.244.190/qna/t/helm-intro-to-chart/5790) 내용을 참고해 작성한 내용입니다.

# Charts

Helm은 패키징을 위한 포맷으로 차트(charts)를 사용합니다. 차트는 쿠버네티스의 리소스들을 생성하기 위해 사용되는 파일들의 모음입니다. 하나의 차트를 이용해서 Memcached의 팟(Pod) 처럼 간단한 것에서 부터 HTTP 서버, 데이터베이스, 캐시 같은 것을 모두 포함한 풀스택 웹 애플리케이션처럼 복잡한 것까지 모두 배포할 수 있습니다.

차트는 관련된 파일들을 특정 디렉토리에 넣어서 만들 수 있는데, 해당 디렉토리를 차트명과 버전을 조합한 이름의 압축 파일로 묶어서 배포할 수 있습니다.

## 차트 파일 구조

차트는 디렉토리에 여러 파일들을 저장해서 관리하고 있습니다. 디렉토리의 이름은 차트의 이름과 동일합니다. (버전을 따로 명시하지는 않습니다.) 예를 들어 워드프레스를 배포하기 위한 차트는 `wordpress/` 디렉토리 내에 저장되어야 합니다.

Helm은 해당 디렉토리 내의 구조를 다음과 같이 가정하고 있습니다.

```yaml
wordpress/
  Chart.yaml          # 차트의 기본정보를 담고 있는 YALM 파일
  LICENSE             # OPTIONAL: 차트의 라이선스에 대해 기술한 텍스트 파일
  README.md           # OPTIONAL: 사람이 읽을 수 있는 README 파일
  requirements.yaml   # OPTIONAL: 차트의 의존성들을 기술한 YALM 파일
  values.yaml         # 차트의 기본적인 설정값들
  charts/             # 현재 차트를 배포하기 위해서 필요한 의존성이 걸려있는 다른 차트들을 저장한 디렉토리
  templates/          # 쿠버네티스의 매니페스트 파일들을 생성하기 위한 탬플릿 파일들
  templates/NOTES.txt # OPTIONAL: 차트의 사용법을 담고 있는 텍스트 파일
```

Helm은 `charts/` 와 `templates/` 디렉토리 내에 있는 파일들 및 위에 나열된 이름의 파일들을 사용합니다. 그 외의 파일들은 무시합니다.

예로써 stable <https://github.com/helm/charts/tree/master/stable/wordpress> 레파지토리에서 관리되는 차트를 한번 살펴보도록하겠습니다.



### Chart.yaml 파일

`Chart.yaml` 파일은 차트를 작성할 때 필수적으로 만들어야 되는 파일입니다. 이 파일에는 다음과 같은 항목이 포함되어 있습니다.

```yaml
apiVersion : (필수) 차트의 API 버전으로 현재는 무조건 "v1" 입니다.
name : (필수) 차트의 이름
version : (필수) 차트 자체의 버전으로 SemVer 2 규격으로 작성합니다.
kubeVersion : (옵션) 이 차트와 호환되는 쿠버네티스의 버전을 SemVer 규격으로 명시할 수 있습니다.
description : (옵션) 이 프로젝트에 대한 한 줄 요약
keywords:
  - (옵션) 이 프로젝트의 키워드들의 리스트
home: (옵션) 프로젝트의 홈페이지 URL
sources:
  - (옵션) 프로젝트의 소스코드 위치의 목록
maintainers: # (옵션)
  - name: (관리자를 넣는 경우 필수) 차트 관리자의 이름
    email: (옵션) 관리자의 메일 주소
    url: (옵션) 관리자의 URL
engine: (옵션) 탬플릿 엔진의 이름 (기본값 : gotpl)
icon: (옵션) 아이콘으로 사용될 수 있는 SVG나 PNG 파일의 URL
appVersion: (옵션) 차트가 배포하는 앱의 버전. SemVer 규격이 아니어도 상관 없음.
deprecated: (옵션) 차트가 deprecate 되었는지 여부 (boolean)
tillerVersion: (옵션) 차트가 설치될 수 있는 Tiller 버전을 SemVer 규격으로 작성합니다.
```

만약 Helm Classic의 `Chart.yaml`을 사용해보신 적이 있다면, 차트의 의존성을 표시하던 부분이 사라진 것에 유의하셔야 합니다. 새로운 차트 포맷은 의존성을 `charts/` 디렉토리에서 관리합니다.

위에 명시되지 않은 다른 필드들은 그냥 무시됩니다.

### Charts 와 Versioning

모든 차트는 반드시 버전을 명시해야 합니다. 버전은 [SemVer 24](http://semver.org/) 규격에 맞춰서 기술되어야 합니다. Helm Classic과 다르게 쿠버네티스 Helm은 버전을 릴리즈에 포함해서 사용하고 있습니다. 어떤 저장소에 위치한 패키지들은 모두 이름과 버전의 조합으로 관리됩니다.

예를 들어 `nginx` 차트에 버전을 `version: 1.2.3`과 같이 기술했다면 저장소에는 다음과 같이 저장됩니다.

```
nginx-1.2.3.tgz
```

SemVer 2 규격에만 맞다면 `version: 1.2.3-alpha.1+ef365` 처럼 더 복잡한 형태의 버전도 사용하실 수 있습니다. 하지만 SemVer 규격에 맞지 않는 버전명은 모두 무시됩니다.

**참고:** Helm 클래식과 Deployment Manager가 GitHub을 기준으로 만들어진 것에 비해서 쿠버네티스 Helm은 GitHub이나 Git에 의존하지 않습니다. 때문에 버전명을 명시할 때 Git SHA를 사용하지도 않습니다.

`Chart.yaml`에 명시된 `version` 필드는 CLI나 Tiller 서버를 포함한 Helm과 관련된 툴에서 광범위하게 사용됩니다. 예를 들어서 패키지를 생성할때 `helm package` 명령을 사용하면 Helm은 `Chart.yaml` 파일에서 버전을 찾아서 패키지명의 일부로 사용합니다. 시스템은 패키지명에 있는 버전과 `Chart.yaml` 파일 내부에 기술된 버전이 일치할 것으로 가정하고 있기 때문에, 만약 둘이 다른 버전을 가진다면 에러를 발생시킵니다.

### appVersion 항목

`appVersion` 항목은 `version` 항목과 크게 관련이 없습니다. 해당 항목은 차트 내부에 포함된 애플리케이션의 버전을 의미합니다. 예를 들어서 `drupal` 차트는 `appVersion: 8.2.1`과 같은 항목을 가질 수 있는데, 이는 chart에 포함된 Drupal의 버전이 8.2.1이라는 것을 나타냅니다. 이 항목은 사용자에게 정보를 제공하기 위한 목적으로 사용되며 차트 자체의 버전을 표시하거나 계산할 때는 사용되지 않습니다.

### Deprecating Charts

차트 저장소에서 차트를 관리할 때 가끔 차트를 폐기(deprecate) 시켜야 될 때가 있습니다. `Chart.yaml`에서 옵셔널 하게 사용할 수 있는  `deprecated` 필드는 차트가 폐기되었는지 여부를 표시할 때 사용할 수 있습니다. 만약 저장소에 있는 특정 차트의 가장 최신 버전이 폐기된 것으로 표시되어 있다면, 저장소에 있는 해당 차트는 폐기된 것으로 간주됩니다. 하지만 폐기되지 않은 새로운 버전의 차트를 다시 배포한다면 해당 차트는 다시 사용될 수 있습니다. [kubernetes/charts4](https://github.com/kubernetes/charts) 프로젝트에서 차트를 폐기할 때 사용하는 프로세스는 다음과 같습니다. 

- `Chart.yaml` 파일의 deprecated 필드를 true로 표기하고, 버전을 올립니다
- 새 버전의 차트를 저장소에 배포합니다
- git 등의 소스 저장소에서 에서 해당 차트를 제거합니다



## Chart LICENSE, README and NOTES

차트는 설치 방법이나 환결 설정, 사용법, 라이선스등을 명시한 파일을 포함할 수 있습니다. README 파일(README.md)은 마크다운 문법으로 작성되어야 하며, 보통 다음과 같은 내용을 포함합니다.

- 차트가 제공하는 서비스나 애플리케이션에 대한 설명
- 차트를 실행하기 위해서 필요한 사전 조건들
- `values.yaml` 파일에 설정할 수 있는 내용들에 대한 설명 및 기본값
- 그 외 차트의 설치와 설정에 필요한 정보들

차트를 설치한 직후나 릴리즈의 상태를 조회할 때 표시할 내용은 `template/NOTES.txt` 파일에 짧은 평문형태로 추가하실 수 있습니다. 이 파일은 [template](https://docs.helm.sh/developing_charts/#templates-and-values) 문법을 사용할 수 있으며 사용법에 대한 정보나 추가로 해야될 작업들이나 차트 및 릴리즈와 관련된 관련된 이런저런 정보들을 포함할 수 있습니다. 예를 들면 데이터베이스에 접속하는 방법이나 웹 UI에 접속하기 위한 방법들이 포함될 수 있습니다. 이 파일은 `helm install` 명령이나 `helm status` 명령을 실행할 때 표준출력으로 표시되기 때문에 가급적이면 간단하게 작성하고 상세한 내용은 README를 참고하도록 하시는 것이 좋습니다.



## 차트 의존성(Chart Dependencies)

Helm은 하나의 차트는 다른 한 개 혹은 여러 개의 차트와 의존관계(서브차트)를 가질 수 있습니다. 이러한 의존성들은 `requirements.yaml` 파일을 이용해서 동적으로 연결하거나 `charts/` 디렉토리를 이용해서 직접 관리하실 수 있습니다.

일부 팀에서는 의존성을 직접 관리하는 것이 유리할 수도 있지만, 일반적으로 의존성은 차트에 포함된 `requirements.yaml` 파일을 이용해서 관리하는 것이 선호됩니다.

**참고:** Helm Classic의 `Chart.yaml` 파일에서 사용하던 `dependencies:` 항목은 더 이상 사용되지 않습니다.

### `requirements.yaml`을 이용한 의존성 관리

`requirements.yaml` 파일은 차트에 필요한 의존성들을 단순히 나열한 파일입니다.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

- `name` 항목은 필요한 차트의 이름입니다.
- `version` 항목은 필요한 차트의 버전입니다.
- `repository` 항목은 차트 저장소의 URL입니다. 저장소를 추가하기 위해서 `helm repo add` 명령을 사용해야 된다는 사실에 주의하세요.

의존성에 대한 정의를 해당 파일에 명시해두었다면, `helm dependency update` 명령을 통해 해당 서브차트들을 차트의 `charts/` 디렉토리 내에 모두 다운로드 받을 수 있습니다.

```bash
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

`helm dependency update` 명령을 이용해 차트를 다운로드 받으면, 해당 차트들은 `charts/` 디렉토리에 압축된 형태로 저장됩니다. 위의 예제에서는 다음과 같은 형태로 저장될겁니다.

```yaml
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

차트를 `requirements.yaml` 파일로 관리하는 것은 의존관계에 있는 차트들을 최신 버전으로 유지하거나 해당 내용을 팀간에 공유할 때 매우 유용합니다.

### requirements.yaml 파일의 alias 항목

의존성을 정의할 때 위에 설명한 필드 외에도 `alias`라는 항목을 정의하실 수 있습니다.

의존성에 alias를 추가해주면 차트 내에 해당 의존성이 추가될 때 alias에 지정해준 이름으로 저장됩니다.

`alias`를 사용하는 예를 들어보면, 아래와 같은 차트를 각각 다른 이름으로 여러 개 다운로드 받아야 될 때를 생각해볼 수 있습니다.

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

위 예제에서는 `parentchart`에 다음과 같은 3개의 의존성을 다운로드 받을 수 있습니다.`

```
subchart
new-subchart-1
new-subchart-2
```

만약 의존성을 직접 관리하고자 하는 경우에는 `charts/` 디렉토리에 같은 차트를 여러 번 복사/붙여넣기 하신 뒤 다른 이름을 지정해주시면 됩니다.

### requirements.yaml 파일의 tag와 condition 항목

위에서 설명한 항목들 외에도 각각의 의존성 정의에는 필요한 경우 `tags`와 `condition` 항목을 추가하실 수 있습니다.

requirement 파일에 정의된 모든 차트들은 전부 같이 올라오는 것이 기본 설정입니다. 하지만 만약 `tags`나 `condition` 항목이 정의되어 있다면, 해당 차트들은 올라오기 전에 조건에 만족되는지를 먼저 판단합니다.

Condition - condition 항목은 하나 혹은 콤마로 구분된 다수의 YAML 경로로 작성됩니다. 만약 해당 경로가 부모 차트의 밸류에 boolean 형태로 정의된 경우 차트는 해당 값에 따라 활성화 되거나 비활성화 될 수 있습니다. 다수의 YAML 경로가 정의된 경우, 앞에서 부터 판단하여 부모 차트에서 찾을 수 있는 가장 첫번째 경로가 판단 대상이 됩니다. 만약 정의된 모든 경로들을 부모차트에서 찾을 수 없는 경우 condition 항목은 아무런 영향을 주지 않습니다.

Tags - tags 항목은 YAML의 목록 형태로 정의되는 label 입니다. 부모차트의 밸류에서 해당 태그들에 각각 boolean 값을 주어서 서브차트를 활성화 하거나 비활성화 시킬 수 있습니다.

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled,global.subchart1.enabled
    tags:
      - front-end
      - subchart1

  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
      
# parentchart/values.yaml
subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

위 예제에서 `front-end` 태그를 가진 모든 차트들을 비활성화 되도록 정의되어 있으나, `subchart1.enabled` 경로가 'true'로 지정되어 있으므로, 이 컨디션은 `front-end` 태그의 정의를 덮어씌우게 되고, `subchart1`은 활성화 됩니다.

`subchart2`의 경우에는 `back-end` 태그가 붙어있으며, 부모차트의 밸류에서 해당 태그가 `true`로 지정되어 있기 때문에 활성화 됩니다. 여기에서 `subchart2`는 `requirements.yaml`에 컨디션을 지정해두었으나, 밸류에서 해당 경로의 값을 찾을 수 없기 때문에 지정된 컨디션은 해당 차트에 영향을 주지 않습니다.

#### CLI에서 태그/컨디션 사용하기

`--set` 파라메터를 태그나 컨디션을 변경하기 위해서 사용하실 수 있습니다.

```
helm install --set tags.front-end=true --set subchart2.enabled=false
```

#### 태그 및 컨디션 적용 우선순위

- **값이 정의된 경우 컨디션은 언제나 태그보다 우선 적용됩니다.** 부모차트에 존재하는 첫번째 컨디션은 그 이후에 존재하는 모든 컨디션보다 우선 적용되며, 나머지 컨디션은 무시됩니다.
- 서브차트에 지정된 태그 중 하나라도 true가 있다면 해당 차트는 활성화 됩니다.
- 태그와 컨디션은 최상위 부모차트의 밸류에 정의된 값을 기준으로 합니다.
- 밸류에 정의된 `tags:` 키는 반드시 최상위 키로 정의되어야 합니다. 중첩된 `tags`는 지원하지 않습니다.



### requirements.yaml에서 하위 차트의 값 가져오기

어떤 경우에는 하위 차트에 있는 값을 부모 차트에 전달해서 해당 값을 공유되는 값으로 사용하고자 할 때가 있습니다. `exports` 포맷을 사용하는 방식의 또 다른 장점으로는 나중에 만들어질 수 있는 일부 툴들이 사용자가 정의한 값들에 대한 인스펙션을 할 수 있도록 해준다는 것입니다.

참조하고자 하는 하위 차트의 값은 부모 차트의 requirements.yaml에 YAML 목록 형태로 명시할 수 있습니다. 목록에 있는 각각의 값들은 하위 차트의 `exports` 항목에 정의된 값 중 일치하는 값을 가져오게 됩니다.

만약 `exports` 항목에 포함되지 않은 값을 가져오고자 하는 경우 [child-parent](https://docs.helm.sh/developing_charts/#using-the-child-parent-format) 형식을 사용하실 수 있습니다. 두 방식은 모두 아래 예제에서 설명드리겠습니다.

#### exports 포맷 사용하기

만약 하위 차트의 `values.yaml` 파일이 `exports` 항목을 루트에 정의했다면 해당 값들은 아래 예제에서 확인할 수 있는 것 처럼 부모 차트의 밸류에서 해당 키를 명시해서 바로 사용할 수 있습니다.

```yaml
# parent's requirements.yaml file
    ...
    import-values:
      - data
# child's values.yaml file
...
exports:
  data:
    myint: 99
```

부모 차트에서 `data`를 가져오겠다고 명시하였으므로, Helm은 하위 차트 중 `data` 키를 `exports` 필드에 정의한 차트가 있는지 찾아보고 해당 값을 가져옵니다.

부모 차트의 밸류는 최종적으로 다음과 같이 하위 차트의 값을 포함하게 됩니다.

```yaml
# parent's values file
...
myint: 99
```

다만 부모 차트에 최종적으로 정의된 값이 부모키인 `data` 키는 포함되지 않음에 유의하여 주세요. 만약 부모키를 표시하고자 하는 경우 'child-parent' 포맷을 사용하여 주세요.

#### child-parent 포맷 사용하기

하위 차트에 정의된 값 중 `exports` 키에 포함되지 않은 값을 가져오기 위해서는 가져오기 위한 값의 경로를 `child`키로, 해당 값을 추가하기 위한 부모 차트의 경로를 `parent` 키에 명시해야 합니다.

아래 예제에 정의된 `import-values` 키는 Helm에서 `child:`에 정의된 경로를 찾아서 `parent:`경로에 정의된 부모 차트의 경로에 복사하도록 해줍니다.

```yaml
# parent's requirements.yaml file
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

위의 예제에서 subchart1에 정의된 `default.data`의 값은 부모차트의 `myimports` 키로 아래와 같이 추가될 것입니다.

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

부모 차트의 최종적인 값은 아래와 같습니다.

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

이제 부모 차트에서 subchart1에서 가져온 `myint`와 `mybool` 항목을 사용하실 수 있습니다.

### `charts/` 디렉토리를 이용해서 서브차트 직접 관리하기

만약에 서브차트를 더 상세하게 컨트롤하고 싶으시다면 해당 서브차트들을 `charts/` 디렉토리에 명시적으로 집어 넣어서 의존성을 관리하실 수도 있습니다.

이 때 서브차트는 압축 파일 형태(`foo-1.2.3.tgz`)나 압축을 해제한 디렉토리 형태로 사용하실 수 있습니다. 하지만 해당 파일이나 디렉토리의 이름이 `_`이나 `.`으로 시작한다면 차트 로더는 해당 서브차트를 무시합니다.

예를 들어서, 워드프레스 차트가 아파치 차트에 의존성을 가지고 있다고 가정하면, 원하는 버전의 아파치 차트는 워드프레스 차트의 `charts/` 디렉토리에서 참조될 수 있습니다.

```yaml
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

위 예제에서 워드프레스 차트가 의존관계를 가지는 Apache와 MySQL 차트를 `charts/` 디렉토리에서 직접 관리하고 있는 것을 확인하실 수 있습니다.

**TIP:** 원격저장소에 있는 차트를 `charts/` 디렉토리에 다운로드 받고 싶으시다면, `helm fetch` 명령어를 사용하실 수 있습니다.

### 의존성 관리에 대한 조금 더 상세한 내용들

지금까지 어떻게 차트의 의존성을 명시하는지에 대해서 설명했는데, 이것들이 `heml install`이나 `helm upgrade`와 같은 명령어을 만나면 어떻게 동작할까요?

"A" 라는 이름의 차트가 아래와 같은 쿠버네티스 오브젝트들을 생성한다고 가정하겠습니다.

- namespace "A-Namespace"
- statefulset "A-StatefulSet"
- service "A-Service"

그리고 A 차트는 B 차트에 의존성을 가지는데 B 차트는 다음과 같은 오브젝트를 생성한다고 가정하겠습니다.

- namespace "B-Namespace"
- replicaset "B-ReplicaSet"
- service "B-Service"

A 차트를 설치하거나 업그레이드 하면 하나의 Helm 릴리즈가 생성되거나 업데이트 됩니다. 이 릴리즈는 다음과 같은 순서로 쿠버네티스 오브젝트들을 생성하거나 업데이트 합니다.

- A-Namespace
- B-Namespace
- A-StatefulSet
- B-ReplicaSet
- A-Service
- B-Service

이는 Helm이 차트를 설치하거나 업그레이드 할 때, 차트로 인해 생성되는 쿠버네티스 오브젝트들과 서브차트의 오브젝트들은 모두 다음 규칙을 따르기 때문입니다.

- 모두 하나의 집합으로 합칩니다.
- 오브젝트의 타입 순서 및 이름 순서로 정렬합니다.
- 정렬된 순서대로 생성하거나 업데이트 합니다.

이후 차트 및 서브차트에 정의된 모든 오브젝트들이 하나의 릴리즈로 생성됩니다.

쿠버네티스 오브젝트의 타입에 따른 정렬 순서는 kind_sorter.go 파일에 InstallOrder라는 열거형 변수로 정의되어 있습니다. ([Helm 소스파일](https://github.com/kubernetes/helm/blob/master/pkg/tiller/kind_sorter.go#L26)을 참고하세요)



## 탬플릿과 밸류

Helm 차트 탬플릿은 [Go 탬플릿 언어](https://golang.org/pkg/text/template/)로 작성할 수 있으며, [스프리그 라이브러리](https://github.com/Masterminds/sprig)에서 가져온 50개 이상의 탬플릿 함수들과 일부 [특화된 함수들](https://docs.helm.sh/developing_charts/#chart-development-tips-and-tricks)을 사용하실 수 있습니다.

모든 탬플릿 파일들은 각 차트의 `templates/` 디렉토리에 저장됩니다. Helm이 차트를 구성할 때, 해당 디렉토리 내에 있는 모든 파일은 탬플릿 엔진에 전달됩니다.

탬플릿에 밸류는 두 가지 방법으로 제공됩니다

- 차트 개발자는 `values.yaml` 파일을 차트 내부에 포함시킬 수 있습니다. 이 파일은 기본값들을 가지고 있습니다.
- 차트 사용자는 밸류를 포함한 별도의 YAML 파일을 사용할 수 있습니다. 이 파일은 `helm install` 명령을 사용할 때 지정할 수 있습니다.

사용자가 커스텀 값들을 제공하는 경우, 해당 값들은 차트의 `values.yaml`에 정의된 값을 덮어 씌웁니다.

### 탬플릿 파일

탬플릿 파일은 Go 탬플릿의 기본적인 컨벤션을 따라갑니다. (자세한 것은 [Go 언어의 탬플릿 가이드](https://golang.org/pkg/text/template/)를 확인하세요) 탬플릿 파일은 아래 예제처럼 작성될 수 있습니다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

위 예제는 <https://github.com/deis/charts> 에서 가져온 것인데, 쿠버네티스의 복제 컨트롤러에 대한 탬플릿이 어떻게 작성되는지 보여줍니다. 여기에는 다음과 같은 4개의 탬플릿 값이 사용되었습니다. (보통 해당 값들은 `values.yaml`에 정의됩니다)

- `imageRegistry`: Docker 이미지를 저장하고 있는 레지스트리
- `dockerTag`: Docker 이미지의 태그
- `pullPolicy`: 쿠버네티스의 이미지 pull 정책
- `storage`: 데이터베이스 저장소에 대한 값으로 `"minio"`를 기본값으로 함

이 모든 값들은 탬플릿 작성자가 필요에 따라 정의해서 사용할 수 있습니다. Helm 규격에는 반드시 포함해야 되는 값이나 파라메터는 없습니다.

실제로 작성된 차트들을 구경하고 싶으시다면, [Kubernetes Charts project](https://github.com/kubernetes/charts)를 확인해보세요



### 사전정의된 값들

`values.yaml` 파일에 정의되거나 `--set` 플래그를 통해서 전달된 값들은 탬플릿에서 `.Values` 객체를 통해 접근할 수 있습니다. 하지만 그 값들 외에도 탬플릿에서 접근할 수 있는 몇가지 사전정의된 값들이 있습니다.

아래 나열된 값들은 모두 사전정의되어 있으며, 모든 탬플릿에서 사용할 수 있고 다른 값으로 덮어씌울 수 없습니다. 모든 값의 이름은 대소문자를 구분합니다.

- `Release.Name`: 릴리즈의 이름 (차트의 이름이 아닙니다)
- `Release.Time`: 차트가 마지막으로 업데이트 된 시간. 릴리즈 객체의 `Last Released` 시간과 같습니다.
- `Release.Namespace`: 차트가 릴리즈 된 네임스페이스
- `Release.Service`: 릴리즈를 동작시킬 서비스. 보통은 `Tiller` 입니다.
- `Release.IsUpgrate`: 만약 릴리즈가 업그레이드나 롤백을 통해 배포된 경우 true가 전달됩니다
- `Release.IsInstall`: 만약 릴리즈가 설치명령을 통해 배포된 경우 true가 전달됩니다.
- `Release.Revision`: 릴리즈의 리비전 번호. 1부터 시작해서 `helm upgrade` 명령을 수행할 때 마다 1씩 올라갑니다.
- `Chart`: `Chart.yaml` 파일의 내용. 예를 들어서 차트의 버전은 `Chart.Version`으로 접근할 수 있으며, 차트 관리자는 `Chart.Maintainers`와 같이 접근할 수 있습니다.
- `Files`: 맵 형태로 차트내의 일반적인 파일에 접근할 수 있습니다. 이 객체를 이용해서 탬플릿에 접근할 수는 없지만 `.helmignore`에 포함되지 않은 다른 파일들에 대한 접근을 가능하게 해줍니다. 파일은 `{{index .Files "file.name}}` 형태로 접근하거나 `{{.Files.Get name}}` 혹은 `{{.Files.GetString name}}` 함수등을 이용하실 수 있습니다. 또 파일의 `[]byte` 컨텐츠에 접근하기 위해서 `{{.Files.GetBytes}}`를 사용하실 수 있습니다.
- `Capabilities`: 맵 형태의 오브젝트로 쿠버네티스(`{{.Capabilities.KubeVersion}}`), 틸러(`{{.Capabilities.TillerVersion}}`), 쿠버네티스 API버전(`{{.Capabilities.APIVersions.Has "batch/v1"`) 정보를 담고 있습니다.

**참고:** Chart.yaml 파일에 포함된 항목 중 알 수 없는 항목들은 모두 무시됩니다. 때문에 `Chart` 객체를 통해서도 해당 항목들은 접근하실 수 없습니다. 임의의 데이터를 전달하기 위한 용도로 Chart.yaml 파일을 사용하실 수는 없으므로 밸류 파일을 이용해주시기 바라겠습니다.

### Values files

방금 전에 설명한 탬플릿들에서 잠깐씩 언급했었는데, `values.yaml` 파일은 필요한 값들을 다음과 같은 형태로 정의해서 사용합니다.

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

밸류 파일은 YAML 포맷으로 작성합니다. 차트는 보통 기본으로 `values.yaml` 파일을 포함합니다. 다만 사용자는 `Helm install` 명령을 줄 때 자신이 원하는 YAML 파일을 지정해서 차트에서 제공하는 기본 values.yaml 파일을 오버라이드 할 수 있습니다.

```
$ helm install --values=myvals.yaml wordpress
```

이렇게 별도의 밸류 파일을 전달하는 경우, Helm은 차트의 기본 값들과 전달받은 값들을 하나로 합칩니다. 예를 들어서 `myvals.yaml` 파일이 아래와 같이 지정되어 있다고 가정하겠습니다.

```yaml
storage: "gcs"
```

만약 이 파일을 위에서 보여드린 차트의 `values.yaml` 파일과 합친다면, 결과적으로 사용되는 값들은 다음과 같은 형태가 될 것입니다.

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

마지막 항목만 재정의 되었음을 알 수 있습니다.

**참고:** 차트에 포함되는 기본값들은 반드시 `values.yaml`이라는 이름으로 저장되어야 합니다. 명령어를 통해 전달되는 밸류 파일의 이름은 무엇이든 상관 없습니다.

**참고:** 만약 `--set` 플래그를 `helm install` 이나 `helm upgrade` 명령어와 같이 사용하신다면 이 값들은 클라이언트에 의해서 YAML 형태로 변환될 것입니다.

**참고:** 만약에 밸류 파일에 필수적으로 정의되어야 되는 값이 있다면, 탬플릿의 ['required' 함수](https://docs.helm.sh/developing_charts/#chart-development-tips-and-tricks)를 이용해서 명시하실 수 있습니다.

밸류 파일에 정의한 모든 값들은 `.Values` 객체를 통해 탬플릿 내에서 사용하실 수 있습니다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

### 스코프, 의존성, 밸류

밸류 파일을 이용하여 최상위 차트의 값을 정의할 수 있는데, 이 때 해당 차트의 `charts/` 디렉토리 내에 있는 모든 차트의 값들이 포함됩니다. 다르게 말하자면 밸류 파일은 자신의 차트와 자신에게 의존성을 가지는 차트들에게 값들을 제공해줍니다. 예를 들어서 워드프레스 차트가 `mysql`과 `apache` 차트에 의존성을 가지고 있다고 가정하겠습니다. 밸류 파일은 해당 차트들에게 값을 전달해줍니다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

가장 위쪽에 위치한 차트는 아래에 위치한 모든 값들에 접근할 수 있습니다. 따라서 워드프레스 차트는 MySQL의 비밀번호를 `.Values.mysql.password`와 같이 사용할 수 있습니다. 하지만 아래쪽에 위치한 차트는 그 상위 차트에 정의된 값에 접근할 수 없습니다. 그래서 MySQL은 `title` 값에 접근할 수 없습니다. 마찬가지로 `apache.port`에도 접근할 수 없습니다.

값들은 네임스페이스를 가지고 있지만, 네임스페이스는 생략될 수 있습니다. 그래서 워드프레스 차트는 MySQL의 비밀번호를 `.Values.mysql.password`로 접근 가능하지만 MySQL 차트에서는 밸류의 스코프를 생략하고 네임스페이스를 제거하여 표기할 수 있기 때문에 비밀번호를 `.Values.password`와 같이 접근할 수 있습니다.

#### Global Values

Helm은 2.0.0-Alpha.2 버전부터 글로벌 밸류를 지원하기 시작했습니다. 방금 설명드린 예제파일을 다음과 같이 수정했다고 하겠습니다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

위에서 `global` 섹션을 추가하고 `app: MyWordPress` 항목을 정의했습니다. 이 값은 모든 차트에서 `.Values.global.app`으로 사용할 수 있습니다.

예를 들어서 `mysql` 탬플릿은 `app`에 `{{.Values.global.app}}`과 같이 접근할 수 있고, `apache` 차트도 같은 방식으로 접근 가능합니다. 조금 더 효과적으로 아래와 같이 다시 작성할 수도 있습니다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

이렇게 작성하면 최상위에 위치한 값을 모든 서브 차트에 전달하실 수 있는데, `metadata`의 label과 같은 항목을 설정할 때 유용합니다.

만약 서브차트가 글로벌 밸류를 정의할 경우, 해당 값은 아래쪽(서브차트의 서브차트)으로 전달됩니다. 하지만 상위 차트로 전달되지는 않습니다. 서브차트가 상위 차트에 값을 전달할 수 있는 방법은 없습니다.

또한 상위차트에 정의된 글로벌 밸류는 항상 하위차트에서 전달된 글로벌 밸류보다 우선시 됩니다.

### 참고자료

탬플릿이나 밸류 파일을 작성할 때, 작성 표준에 대해 도움이 될 수 있는 참고 사이트는 아래와 같습니다.

- [Go 탬플릿](https://godoc.org/text/template)
- [기타 탬플릿 함수들](https://godoc.org/github.com/Masterminds/sprig)
- [YAML 스펙](http://yaml.org/spec/)

위 문서의 마크다운 원문은 [Github](https://code.sdsdev.co.kr/jheo/peachtree/blob/master/guide/helm_chart_guide.md) 에 올려두었습니다. PDF 파일은 [여기](https://code.sdsdev.co.kr/jheo/peachtree/blob/master/guide/helm_chart_guide.pdf)에서 받으실 수 있습니다.



## Using Helm to Manage Charts

The `helm` tool has several commands for working with charts.

It can create a new chart for you:

```
$ helm create mychart
Created mychart/
```

Once you have edited a chart, `helm` can package it into a chart archive for you:

```
$ helm package mychart
Archived mychart-0.1.-.tgz
```

You can also use `helm` to help you find issues with your chart's formatting or information:

```
$ helm lint mychart
No issues found
```


