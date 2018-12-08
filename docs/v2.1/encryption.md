---
title: Encryption at Rest
summary: Encryption at Rest encrypts node data on local disk transparently.
toc: true
---

<span class="version-tag">v2.1의 새로운 기능:</span>
Rest에서의 암호화는 로컬 디스크에 있는 노드의 데이터를 투명하게 암호화한다.

{{site.data.alerts.callout_danger}}
**이는 실험적인 기능입니다.**  버그 또는 사용자 오류의 경우, 암호화된 노드 스토어의 모든 데이터는 사용할 수 없게 될 수 있다. 실험 상태를 벗어날 때까지 프로덕션 데이터에 대한 Rest에서의 암호화를 사용하지 마십시오.  그때까지, 이 기능은 테스트 환경에서만 사용해야 합니다.
<br />
<br />
버그가 발생하면, [문제 제기](file-an-issue.html)하십시오. 
{{site.data.alerts.end}}

## 개요


Rest에서의 암호화는 모든 키 크기가 허용되는 [카운터 모드](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_(CTR))의 [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)를 사용하여 디스크에 있는 모든 파일을 암호화할 수 있다.

암호화는 [저장소 계층](architecture/storage-layer.html)에서 수행되고 저장소별로 구성됩니다.
내용에 관계없이 상점에서 사용하는 모든 파일은 원하는 알고리즘으로 암호화됩니다.

임의의 순환 스케쥴을 허용하고 키의 보안을 보장하기 위해, 두 개의 키 계층을 사용합니다:

+ **키 저장**은 사용자에 의해 파일로 제공합니다. 이들은 데이터 키 목록을 암호화하는 데 사용됩니다 (아래 참조). 이것을 **키 암호화 키**라고 합니다: 유일한 목적은 다른 키를 암호화하는 것입니다. 저장소 키는 CockroachDB에 의해 절대 유지되지 않습니다. 이 키를 사용하여 암호화된 데이터는 매우 적기 때문에, 재사용 위험없이 매우 긴 수명을 유지할 수 있습니다.

+ **데이터 키**는 CockroachDB에 의해 자동 생성됩니다. 이들은 디스크의 모든 파일을 암호화하는 데 사용됩니다.  이것을 **데이터 암호화 키**라고 합니다. 데이터 키는 저장소 키를 사용하여 암호화된 키 레지스트리 파일에 유지됩니다. 키의 재사용을 피하기 위해 수명이 짧습니다.

저장소 키는 노드가 시작할 때 로컬로 읽을 수 있는 파일에 경로를 전달하여 지정합니다. 파일에는 32 바이트 (키 ID)와 그 뒤에 키 (16, 24 또는 32 바이트)가 있어야 합니다. 키 크기에 따라 사용할 AES 버전 (AES-128, AES-192 또는 AES-256)이 결정됩니다. 저장소 키를 만드는 방법을 보여주는 예제는 아래의 [키 파일 생성](#generating-key-files)을 참조하십시오.

또한 노드 시작 중에, CockroachDB는 저장소 키와 동일한 길이의 데이터 키를 사용합니다. 암호화가 활성화되었거나, 키 크기가 변경되었거나, 데이터 키가 너무 오래되면 (기본 수명은 1주), CockroachDB는 새로운 데이터 키를 생성합니다.

저장소에서 생성된 새 파일은 현재-활성 데이터 키를 사용합니다. 모든 데이터 키 (활성 및 이전 키 모두)는 키 레지스트리 파일에 저장되고 활성 저장 키로 암호화됩니다.

시작한 후, 활성 데이터 키가 너무 오래되면, CockroachDB는 새로운 데이터 키를 생성하고 이를 모든 추가 암호화에 사용하여 활성으로 표시합니다.

CockroachDB는 현재 오래된 파일의 재 암호화를 강제하지 않지만, 대신 정상적인 RocksDB churn에 의존하여 모든 데이터를 원하는 암호화로 천천히 다시 쓰게됩니다.

## 키 회전

여러 이유로 인해 Rest에서의 암호화에는 키 회전이 필요합니다:

* 동일한 암호화 매개변수로 키 재사용을 방지합니다.(많은 파일을 암호화한 후)
* 키 노출 위험을 줄이기 위해. 

저장소 키는 사용자에 의해 지정되므로 다른 키를 지정하여 회전되어야 합니다.
이것은 `--enterprise-encryption` 플래그의 `key` 매개변수를 새로운 키의 경로로 설정하고, 이전에 사용된 키를`old-key`로 설정함으로써 이루어집니다.

다음 조건 중 하나가 충족되면 시작시 데이터 키가 자동으로 회전됩니다:

* 활성 저장소 키가 변경되었습니다.
* 암호화 유형이 변경되었습니다 (다른 키 크기 또는 암호화로/암호화에서 일반 텍스트).
* 현재 데이터 키는 `rotation-period` 이상입니다.

현재 데이터 키가 `rotation-period` 이상인 경우, 런타임시 데이터 키가 자동으로 회전됩니다.

한 번 회전하면, 이전 저장소 키를 다시 활성 키로 만들 수 없습니다.

저장소 키 회전시 데이터 키 레지스트리는 이전 키를 사용하여 암호 해독되고 새 키로 암호화됩니다
새로 생성된 데이터 키는 이 지점의 모든 새 데이터를 암호화하는 데 사용됩니다.

## 암호화 유형 변경

사용자는 암호화 유형을 일반 텍스트에서 다른 암호화 알고리즘 사이의 암호화로 변경할 수 있습니다 (다양한 키 크기 사용) 또는 암호화에서 일반 텍스트로 변환할 수 있습니다.

암호화 유형을 일반 텍스트로 변경하면, 데이터 키 레지스트리가 더 이상 암호화되지 않고 이전의 모든 데이터 키는 누구나 읽을 수 있습니다. 저장소의 모든 데이터를 효과적으로 읽을 수 있습니다.

일반 텍스트에서 암호화로 변경하면, 결국 모든 데이터가 다시 쓰여지고 암호화 될 때까지 약간의 시간이 걸립니다.

## 권장 사항

암호화로 실행할 때 유의해야 할 사항이 많이 있습니다.

### 배포 구성

키 누출을 방지하려면, 프로덕션 배포가 다음을 수행해야 합니다:

* 암호화된 스왑을 사용하거나, 스왑을 완전히 비활성화하십시오.
* 코어 파일을 비활성화하십시오.

CockroachDB는 암호화가 요청될 때 시작시 코어 파일을 비활성화하려고 시도하지만, 실패할 수 있습니다.

### 키 핸들링

키 관리는 암호화의 가장 위험한 측면입니다. 다음 규칙을 염두에 두어야 합니다:

* `cockroach` 프로세스를 실행하는 UNIX 사용자만 키에 접근할 수 있는지 확인하십시오.
* CockroachDB 데이터와 동일한 파티션/드라이브에 키를 저장하지 마십시오. 별도의 시스템 (예: [Keywhiz](https://square.github.io/keywhiz/), [Vault](https://www.hashicorp.com/products/vault))에서 실행 시간에 키를 로드하는 것이 가장 좋습니다.
* 저장소 키를 자주 회전 (몇 주에서 몇 달 간격으로).
* 데이터 키 순환 기간을 낮게 유지 (기본값은 1주).

### 다른 권장사항

최상의 시큐리티 관행에는 몇 가지 다른 권장 사항이 적용됩니다:

* 암호화된 데이터를 일반 텍스트로 전환하지 마십시오. 데이터 키가 유출됩니다. 일반 텍스트를 선택하면, 이전에 암호화된 모든 데이터를 도달할 수 있는 것으로 간주해야 합니다.
* 데이터 키를 쉽게 사용할 수 없으므로, 암호화된 파일을 복사하지 마십시오.
* 암호화가 필요한 경우, 일반 텍스트로 실행하지 않고, 첫 번째 실행에서 활성화된 노드를 시작합니다.

{{site.data.alerts.callout_danger}}
Rest에서의 암호화가 활성화되어 있어도, [`BACKUP`](backup.html) 명령문으로 수행된 백업은 **암호화되지 않습니다**. Rest에서의 암호화는 로컬 디스크에서의 CockroachDB 노드의 데이터에만 적용됩니다. 암호화된 백업을 원하면, 원하는 암호화 방법을 사용하여 백업 파일을 암호화해야 합니다.
{{site.data.alerts.end}}

## 예제

### 키 파일 생성

Cockroach는 키 파일의 크기에 따라 사용할 암호화 알고리즘을 결정합니다.
키 파일은 키 ID (32 바이트)와 실제 키 (암호화 알고리즘에 따라 16, 24 또는 32 바이트)를 구성하는 임의의 데이터를 포함해야 합니다.

| 알고리즘 | 키 크기 | 키 파일 크기 |
|-|-|-|
| AES-128 | 128 bits (16 bytes) | 48 bytes |
| AES-192 | 192 bits (24 bytes) | 56 bytes |
| AES-256 | 256 bits (32 bytes) | 64 bytes |

키 파일 생성은 `cockroach` CLI를 사용하여 수행할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen encryption-key -s 128 /path/to/my/aes-128.key
~~~

또는 동등한 [openssl](https://www.openssl.org/docs/man1.0.2/apps/openssl.html) CLI 명령:

{% include copy-clipboard.html %}
~~~ shell
$ openssl rand -out /path/to/my/aes-128.key 48
~~~

### 암호화된 노드 시작

`--enterprise-encryption` command line flag. 암호화는`--enterprise-encryption` 명령 행 플래그를 사용하여 노드 시작 시간에 구성됩니다.
플래그는 노드의 스토어 중 하나에 대한 암호화 옵션을 지정합니다. 여러 저장소가 있는 경우,플래그는 각 저장소에 대해 지정되어야 합니다.

플래그는 다음 형식을 취합니다: `--enterprise-encryption=path=<store path>,key=<key file>,old-key=<old key file>,rotation-period=<period>`.

플래그에 허용되는 구성 요소는 다음과 같습니다:

| 구성 요소 | 요구 여부 | 설명 |
|-|-|-|
| `path`            | 요구됨 | 암호화를 적용할 저장소의 경로. |
| `key`             | 요구됨 | 데이터를 암호화할 키 파일의 경로 또는 일반 텍스트의 `plain` |
| `old-key`         | 요구됨 | 데이터가 암호화된 키 파일의 경로 또는 일반 텍스트의 `plain`. |
| `rotation-period` | 선택 | 데이터 키가 자동으로 순환되는 빈도. 기본값 : 1주. |

`key` 와 `old-key` 구성 요소는 **항상** 지정되어야 합니다. 암호화 알고리즘 간 및 일반 텍스트와 암호화 간 전환을 허용합니다.

AES-128 암호화를 사용하여 처음으로 노드를 시작하는 작업은 다음을 사용하여 수행할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --store=cockroach-data --enterprise-encryption=path=cockroach-data,key=/path/to/my/aes-128.key,old-key=plain
~~~

{{site.data.alerts.callout_danger}}
주어진 저장소에 대해 지정되면, `--enterprise-encryption` 플래그가 항상 존재해야 합니다.
{{site.data.alerts.end}}

### 암호화 상태 확인

암호화 상태는 노드의 저장소 레포트에서 확인할 수 있으며, 다음을 통해 확인할 수 있습니다: `http(s)://nodeaddress:8080/#/reports/stores/local` (또는 `local`을 노드 ID로 대체). 예를 들어, [로컬 클러스터](secure-a-cluster.html)를 실행중인 경우, <https://localhost:8888/#/reports/stores/local>에서 노드의 저장소 보고서를 볼 수 있습니다.

이 레포트에는 다음을 포함하여 선택한 노드의 모든 저장소에 대한 암호화 상태가 표시됩니다:

* 활성 저장소 키 정보.
* 활성 데이터 키 정보.
* 활성 데이터 키를 사용하여 암호화된 파일/바이트 비율.

CockroachDB는 RocksDB 압축에 의존하여 최신 암호화 키를 사용하여 새 파일을 작성합니다. 모든 파일을 교체하는 데 며칠이 걸릴 수 있습니다. 일부 파일은 시작시에만 다시 쓰여지고, 일부는 오래된 복사본을 보관하며, 여러 번 다시 시작해야 합니다. `cockroach debug compact` 명령으로 RocksDB 압축을 강제할 수 있습니다 (노드는 먼저 [중지](stop-a-node.html)되어야 합니다)


각 파일에 사용되는 암호화 키의 자세한 목록은 `cockroach debug encryption-status` 명령을 사용하여 구할 수 있습니다.

키에 대한 정보는 다음을 포함하여 [로그](debug-and-error-logs.html)에 기록됩니다:

* 시작시 활성/이전 키 정보.
* 데이터 키 회전 후 새 키 정보.

### 암호화 알고리즘 또는 키 변경

암호화 유형 및 키는 노드를 다시 시작하여 언제든지 변경할 수 있습니다.
키 또는 암호화 유형을 변경하려면, `--enterprise-encryption` 플래그의 `key` 구성 요소가 새로운 키로 설정되고, 이전에 사용된 키는 `old-key` 구성 요소에 지정되어야 합니다.

예를 들어, 다음을 사용하여 AES-128에서 AES-256으로 전환할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --store=cockroach-data --enterprise-encryption=path=cockroach-data,key=/path/to/my/aes-256.key,old-key=/path/to/my/aes-128.key
~~~

시작시, 노드는 기존의 암호화 키 (`aes-128.key`)를 사용하여 기존 데이터 키를 읽고, 새 키(`aes-256.key`)를 사용하여 데이터 키를 다시 작성합니다. 새로운 데이터 키는 원하는 AES-256 알고리즘과 일치하도록 생성됩니다.

새 키가 활성 상태인지 확인하려면, Admin UI의 스토어 레포트 페이지를 사용하여 [암호화 상태를 확인](#checking-encryption-status)합니다. 

암호화를 비활성화하려면, `key = plain`을 지정하십시오. 데이터 키는 일반 텍스트로 저장되며 새 데이터는 암호화되지 않습니다.
키를 회전하려면, `key=/path/to/my/new-aes-128.key` 및 `old-key=/path/to/my/old-aes-128.key`를 지정하십시오. 데이터 키는 이전 키를 사용하여 암호 해독된 다음 새 키를 사용하여 암호화됩니다. 새로운 데이터 키도 생성됩니다.

## 더 보기

+ [엔터프라이즈 라이센싱](enterprise-licensing.html)
+ [`BACKUP`](backup.html)
+ [노드 시작](start-a-node.html)
