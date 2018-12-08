---
title: Encryption at Rest
summary: Encryption at Rest encrypts node data on local disk transparently.
toc: true
---

<span class="version-tag">New in v2.1:</span>
Rest에서의 암호화는 로컬 디스크에 있는 노드의 데이터를 투명하게 암호화한다.

{{site.data.alerts.callout_danger}}
**이는 실험적인 기능입니다.**  버그 또는 사용자 오류의 경우, 암호화된 노드 스토어의 모든 데이터는 사용할 수 없게 될 수 있다. 실험 상태를 벗어날 때까지 프로덕션 데이터에 대한 Rest에서의 암호화를 사용하지 마십시오.  그때까지, 이 기능은 테스트 환경에서만 사용해야 합니다.
<br />
<br />
버그가 발생하면, [문제 제기](file-an-issue.html)하십시오. 
{{site.data.alerts.end}}

## 개요

Encryption at Rest allows encryption of all files on disk using [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) in [counter mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_(CTR)), with all key
sizes allowed.

암호화는 [저장소 계층](architecture/storage-layer.html)에서 수행되고 저장소별로 구성됩니다.
내용에 관계없이 상점에서 사용하는 모든 파일은 원하는 알고리즘으로 암호화됩니다.

임의의 순환 스케쥴을 허용하고 키의 보안을 보장하기 위해, 두 개의 키 계층을 사용합니다:

+ **키 저장**은 사용자에 의해 파일로 제공합니다. 이들은 데이터 키 목록을 암호화하는 데 사용됩니다 (아래 참조). 이것을 **키 암호화 키**라고 합니다: its only purpose is to encrypt other keys. Store keys are never persisted by CockroachDB. Since very little data is encrypted using this key, it can have a very long lifetime without risk of reuse.

+ **데이터 키**는 CockroachDB에 의해 자동 생성됩니다. 이들은 디스크의 모든 파일을 암호화하는 데 사용됩니다.  이것을 **데이터 암호화 키**라고 합니다. 데이터 키는 저장소 키를 사용하여 암호화된 키 레지스트리 파일에 유지됩니다. 키의 재사용을 피하기 위해 수명이 짧습니다.

Store keys are specified at node startup by passing a path to a locally readable file. The file must contain 32 bytes (the key ID) followed by the key (16, 24, or 32 bytes). The size of the key dictates the version of AES to use (AES-128, AES-192, or AES-256). For an example showing how to create a store key, see [Generating key files](#generating-key-files) below.

Also during node startup, CockroachDB uses a data key with the same length as the store key. If encryption has just been enabled,
the key size has changed, or the data key is too old (default lifetime is one week), CockroachDB generates a new data key.

저장소에서 생성된 새 파일은 현재-활성 데이터 키를 사용합니다. 모든 데이터 키 (활성 및 이전 키 모두)는 키 레지스트리 파일에 저장되고 활성 저장 키로 암호화됩니다.

시작한 후, 활성 데이터 키가 너무 오래되면, CockroachDB는 새로운 데이터 키를 생성하고 이를 모든 추가 암호화에 사용하여 활성으로 표시합니다.

CockroachDB does not currently force re-encryption of older files but instead relies on normal RocksDB churn to slowly rewrite all data with the desired encryption.

## 회전 키

여러 이유로 인해 Rest에서의 암호화에는 키 회전이 필요합니다:

* To prevent key reuse with the same encryption parameters (after encrypting many files).
* To reduce the risk of key exposure.

Store keys are specified by the user and must be rotated by specifying different keys.
This is done by setting the `key` parameter of the `--enterprise-encryption` flag to the path to the new key,
and `old-key` to the previously-used key.

Data keys will automatically be rotated at startup if any of the following conditions are met:

* The active store key has changed.
* The encryption type has changed (different key size, or plaintext to/from encryption).
* The current data key is `rotation-period` old or more.

Data keys will automatically be rotated at runtime if the current data key is `rotation-period` old or more.

Once rotated, an old store key cannot be made the active key again.

Upon store key rotation the data keys registry is decrypted using the old key and encrypted with the new
key. The newly-generated data key is used to encrypt all new data from this point on.

## 암호화 유형 변경

The user can change the encryption type from plaintext to encryption, between different encryption algorithms
(using various key sizes), or from encryption to plaintext.

When changing the encryption type to plaintext, the data key registry is no longer encrypted and all previous
data keys are readable by anyone. All data on the store is effectively readable.

When changing from plaintext to encryption, it will take some time for all data to eventually be re-written
and encrypted.

## 권장 사항

암호화로 실행할 때 유의해야 할 사항이 많이 있습니다.

### 배포 구성

키 누출을 방지하려면, 프로덕션 배포가 다음을 수행해야 합니다:

* 암호화된 스왑을 사용하거나, 스왑을 완전히 비활성화하십시오.
* 코어 파일을 비활성화하십시오.

CockroachDB는 암호화가 요청될 때 시작시 코어 파일을 비활성화하려고 시도하지만, 실패할 수 있습니다.

### 키 핸들링

키 관리는 암호화의 가장 위험한 측면입니다. 다음 규칙을 염두에 두어야 합니다:

* Make sure that only the UNIX user running the `cockroach` process has access to the keys.
* Do not store the keys on the same partition/drive as the CockroachDB data. It is best to load keys at run time from a separate system (e.g., [Keywhiz](https://square.github.io/keywhiz/), [Vault](https://www.hashicorp.com/products/vault)).
* Rotate store keys frequently (every few weeks to months).
* Keep the data key rotation period low (default is one week).

### 다른 권장사항

A few other recommendations apply for best security practices:

* Do not switch from encrypted to plaintext, this leaks data keys. When plaintext is selected, all previously encrypted data must be considered reachable.
* Do not copy the encrypted files, as the data keys are not easily available.
* If encryption is desired, start a node with it enabled from the first run, without ever running in plaintext.

{{site.data.alerts.callout_danger}}
Note that backups taken with the [`BACKUP`](backup.html) statement **are not encrypted** even if Encryption at Rest is enabled. Encryption at Rest only applies to the CockroachDB node's data on the local disk. If you want encrypted backups, you will need to encrypt your backup files using your preferred encryption method.
{{site.data.alerts.end}}

## 예제

### 키 파일 생성

Cockroach determines which encryption algorithm to use based on the size of the key file.
The key file must contain random data making up the key ID (32 bytes) and the actual key (16, 24, or 32
bytes depending on the encryption algorithm).

| Algorithm | Key size | Key file size |
|-|-|-|
| AES-128 | 128 bits (16 bytes) | 48 bytes |
| AES-192 | 192 bits (24 bytes) | 56 bytes |
| AES-256 | 256 bits (32 bytes) | 64 bytes |

Generating a key file can be done using the `cockroach` CLI:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach gen encryption-key -s 128 /path/to/my/aes-128.key
~~~

Or the equivalent [openssl](https://www.openssl.org/docs/man1.0.2/apps/openssl.html) CLI command:

{% include copy-clipboard.html %}
~~~ shell
$ openssl rand -out /path/to/my/aes-128.key 48
~~~

### 암호화된 노드 시작

Encryption is configured at node start time using the `--enterprise-encryption` command line flag.
플래그는 노드의 스토어 중 하나에 대한 암호화 옵션을 지정합니다. 여러 저장소가 있는 경우,플래그는 각 저장소에 대해 지정되어야 합니다.

The flag takes the form: `--enterprise-encryption=path=<store path>,key=<key file>,old-key=<old key file>,rotation-period=<period>`.

플래그에 허용되는 구성 요소는 다음과 같습니다:

| 구성 요소 | Requirement | 설명 |
|-|-|-|
| `path`            | Required | Path of the store to apply encryption to. |
| `key`             | Required | Path to the key file to encrypt data with, or `plain` for plaintext. |
| `old-key`         | Required | Path to the key file the data is encrypted with, or `plain` for plaintext. |
| `rotation-period` | Optional | How often data keys should be automatically rotated. Default: one week. |

The `key` and `old-key` components must **always** be specified. They allow for transitions between
encryption algorithms, and between plaintext and encrypted.

AES-128 암호화를 사용하여 처음으로 노드를 시작하는 작업은 다음을 사용하여 수행할 수 있습니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start --store=cockroach-data --enterprise-encryption=path=cockroach-data,key=/path/to/my/aes-128.key,old-key=plain
~~~

{{site.data.alerts.callout_danger}}
주어진 저장소에 대해 지정되면, `--enterprise-encryption` 플래그가 항상 존재해야 합니다.
{{site.data.alerts.end}}

### 암호화 상태 확인

Encryption status can be seen on the node's stores report, reachable through: `http(s)://nodeaddress:8080/#/reports/stores/local` (or replace `local` with the node ID). For example, if you are running a [local cluster](secure-a-cluster.html), you can see the node's stores report at <https://localhost:8888/#/reports/stores/local>.

이 레포트에는 다음을 포함하여 선택한 노드의 모든 저장소에 대한 암호화 상태가 표시됩니다:

* 활성 저장소 키 정보.
* 활성 데이터 키 정보.
* 활성 데이터 키를 사용하여 암호화된 파일/바이트 비율.

CockroachDB relies on RocksDB compactions to write new files using the latest encryption key. It may take several days for all files to be replaced. Some files are only rewritten at startup, and some keep older copies around, requiring multiple restarts. You can force RocksDB compaction with the `cockroach debug compact` command (the node must first be [stopped](stop-a-node.html)).

A more detailed list of encryption keys in use for each file is available using the `cockroach debug encryption-status` command.

키에 대한 정보는 다음을 포함하여 [로그](debug-and-error-logs.html)에 기록됩니다:

* Active/old key information at startup.
* New key information after data key rotation.

### 암호화 알고리즘 또는 키 변경

Encryption type and keys can be changed at any time by restarting the node.
To change keys or encryption type, the `key` component of the `--enterprise-encryption` flag is set to the new key,
while the key previously used must be specified in the `old-key` component.

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
