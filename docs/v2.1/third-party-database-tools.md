---
title: Third-Party Database Tools
summary: Learn about third-party software that interoperates with CockroachDB.
toc: true
---

이것은 포스트그레스와 호환되는 [드라이버](build-an-app-with-cockroachdb.html), [ORMs](build-an-app-with-cockroachdb.html) 및 기타 타입의 제 3자 데이터베이스 툴을 CockroachDB와 함께 사용할 수 있게 합니다. 일부 툴은 현재로서는 부분적으로만 지원되지만, 다른 툴은 철저히 검사 및 테스트되었습니다.

<span class="version-tag">새로운 버전 2.1:</span> [DBeaver 데이터베이스 도구][dbeaver]는 우리가 전폭적으로 지원한다고 주장하는 툴 중 하나입니다.

[DBeaver website][dbeaver]에 따르면:

> DBeaver는 개발자, SQL 프로그래머, 데이터베이스 관리자 및 분석가를 위한 크로스 플랫폼 데이터베이스 GUI 툴입니다.

이 튜토리얼에서는 시큐어한 CockroachDB 클러스터와 함께 DBeaver를 사용하는 과정을 살펴보겠습니다.

{{site.data.alerts.callout_success}}
DBeaver 사용에 대한 자세한 내용은 [DBever 설명서](https://dbeaver.io/docs/)를 참조하십시오.

문제가 발생하면 [DBeaver 이슈 트래커](https://github.com/dbeaver/dbeaver/issues)에서 문제를 신고하십시오.
{{site.data.alerts.end}}

## 시작하기 전에

이 튜토리얼을 진행하려면 다음 단계를 수행하십시오:

- [CockroachDB 설치](install-cockroachdb.html) 하고 [시큐어 클러스터 시작](secure-a-cluster.html) 합니다.
- [DBeaver](https://dbeaver.io/download/) 버전 5.2.3 이상의 사본을 다운로드하십시오.

1단계. DBeaver를 시작하고 CockroachDB에 연결하십시오.

DBeaver를 시작하고 메뉴에서 **Database > New Connection** 을 선택하십시오. 나타나는 대화 상자에서 목록에서 **CockroachDB**를 선택하십시오.

<img src="{{ 'images/v2.1/dbeaver-01-select-cockroachdb.png' | relative_url }}" alt="DBeaver - select CockroachDB" style="border:1px solid #eee;max-width:100%" />

## 2단계. 연결 설정 업데이트

**Create new connection** 대화 상자가 나타나면 **Network settings**을 클릭하십시오:

<img src="{{ 'images/v2.1/dbeaver-02-cockroachdb-connection-settings.png' | relative_url }}" alt="DBeaver - CockroachDB connection settings" style="border:1px solid #eee;max-width:100%" />

네트워크 설정에서 **SSL**탭을 클릭하십시오. 아래 스크린샷처럼 보일 것입니다.

<img src="{{ 'images/v2.1/dbeaver-03-ssl-tab.png' | relative_url }}" alt="DBeaver - SSL tab" style="border:1px solid #eee;max-width:100%" />

다음과 같이 **Use SSL** 체크 박스를 체크하고 텍스트 영역을 채우십시오:

- **Root certificate**: 안전한 클러스터를 위해 생성한 `ca.crt` 파일을 사용하십시오.
- **SSL certificate**: 클러스터의 루트 인증서에서 생성된 클라이언트 인증서를 사용하십시오. 루트 사용자의 경우, 이 이름은`client.root.crt`로 명명됩니다. 추가 보안을 위해 DBeaver와 함께 사용하기 위해 새 데이터베이스 사용자 및 클라이언트 인증서를 만들 수 있습니다.
- **SSL certificate key**:  DBeaver는 Java 응용 프로그램이기 때문에 아래에 표시된 것과 같이 [OpenSSL 명령](https://wiki.openssl.org/index.php/Command_Line_Utilities#pkcs8_.2F_pkcs5)을 사용하여 키 파일을 `*.pk8` 형식으로 변환해야 합니다. 
파일을 생성했으면 여기에 위치를 입력하십시오. 이 예제에서 파일 이름은 `client.root.pk8`입니다.
    {% include copy-clipboard.html %}
    ~~~ console
    $ openssl pkcs8 -topk8 -inform PEM -outform DER -in client.root.key -out client.root.pk8 -nocrypt
    ~~~

**SSL mode** 드롭 다운에서 **require** 을 선택하십시오. **SSL Factory**를 설정할 필요가 없으며, DBeaver가 기본값을 사용하도록 할 수 있습니다.


## Step 3. 연결 설정 테스트

**Test Connection...** 을 클릭하십시오. 모든 것이 작동하면 아래에 표시된 것과 같은 **Success** 대화 상자가 나타납니다.

<img src="{{ 'images/v2.1/dbeaver-04-connection-success-dialog.png' | relative_url }}" alt="DBeaver - connection success dialog" style="border:1px solid #eee;max-width:100%" />

## 4단계. DBeaver 사용 시작

**Finish**를 클릭하여 CockroachDB와 함께 DBeaver를 시작하십시오.

<img src="{{ 'images/v2.1/dbeaver-05-movr.png' | relative_url }}" alt="DBeaver - CockroachDB with the movr database" style="max-width:100%" />

DBeaver사용에 대한 자세한 내용은 [DBever 설명서](https://dbeaver.io/docs/)를 참조하십시오.

문제가 발생하면 [DBeaver 이슈 트래커](https://github.com/dbeaver/dbeaver/issues)에서 문제를 신고하십시오.

## 더보기

+ [DBeaver 설명서](https://dbeaver.io/docs/)
+ [DBeaver 이슈 트래커](https://github.com/dbeaver/dbeaver/issues)
+ [CockroachDB SQL 배우기](learn-cockroachdb-sql.html)

<!-- Reference Links -->

[dbeaver]: https://dbeaver.io
