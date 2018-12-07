---
title: Manage Users
summary: To create and manage your cluster's users (which lets you control SQL-level privileges), use the cockroach user command with appropriate flags.
toc: true
---

SQL 수준 [권한](privileges.html)을 제어할 수 있는 클러스터 사용자를 생성, 관리 및 제거하려면 적절한 플래그로 `cockroach user` [명령](cockroach-commands.html)을 사용하십시오.

{{site.data.alerts.callout_success}}또한 <a href="create-user.html"><code>CREATE USER</code></a> 및 <a href="drop-user.html"><code>DROP USER</code></a> 명령문을 사용하여 사용자를 생성 및 제거할 수 있습니다.{{site.data.alerts.end}}


## 고려 사항

- 사용자 이름은 대소문자를 구분하지 않으며; 문자나 밑줄로 시작하고; 문자, 숫자 또는 밑줄만 포함해야 하며; 1에서 63자 사이여야 합니다.
- 사용자 생성 후 [데이터베이스 및 테이블에 대한 권한을 부여](grant.html)해야 합니다.
- 모든 사용자는 권한을 [부여](grant.html)하고 [취소](revoke.html)할 수 있는 `public` 역할에 속합니다.
- 시큐어 클러스터에서는 [사용자에 대한 클라이언트 인증서를 작성](create-security-certificates.html#create-the-certificate-and-key-pair-for-a-client)해야 하며 사용자는 [클러스터에 대한 액세스를 인증](#user-authentication)해야 합니다.
- {% include {{ page.version.version }}/misc/remove-user-callout.html %}

## 하위 명령

하위 명령 | 기능
-----------|------
`get` | 사용자와 사용자의 해시 암호가 있는 테이블을 검색합니다.
`ls` | 모든 사용자를 나열합니다.
`rm` | 사용자를 제거합니다.
`set` | 사용자를 생성하거나 업데이트합니다.

## 시놉시스

~~~ shell
# Create a user:
$ cockroach user set <username> <flags>

# List all users:
$ cockroach user ls <flags>

# Display a specific user:
$ cockroach user get <username> <flags>

# View help:
$ cockroach user --help
$ cockroach user get --help
$ cockroach user ls --help
$ cockroach user rm --help
$ cockroach user set --help
~~~

## 플래그

`user` 명령과 하위 명령들은 아래의 [general-use](#general)와 [logging](#logging) 플래그를 지원합니다.

### General

플래그 | 설명
-----|------------
`--password` | 사용자에 대해 비밀번호 인증을 사용 가능하게 합니다; 커맨드 라인에 비밀번호를 입력하라는 메시지가 나타납니다.<br/><br/>비밀번호 생성은 `root`가 아닌 사용자의 시큐어 클러스터에서만 지원됩니다. `root` 사용자는 클라이언트 인증서와 키로 인증해야 합니다.
`--echo-sql` | 커맨드 라인 유틸리티에서 암시적으로 전송된 SQL 문을 나타냅니다. 데모를 보려면 아래의 [예시](#reveal-the-sql-statements-sent-implicitly-by-the-command-line-utility)를 참조하십시오.
`--format` | 표준 출력으로 표기된 테이블 행을 표시하는 방법. 가능한 값: `tsv`, `csv`, `table`, `raw`, `records`, `sql`, `html`.<br><br>**기본값:** [터미널에서 출력](use-the-built-in-sql-client.html#session-and-output-types)되는 세션의 경우 `table`; 그렇지 않은 경우 `tsv`.

### 클라이언트 연결

{% include {{ page.version.version }}/sql/connection-parameters.md %}

자세한 내용은 [클라이언트 연결 매개변수](Connection-parameters.html)를 참조하십시오.

현재 `admin` 역할의 구성원만이 사용자를 생성할 수 있습니다. 기본적으로 `root` 사용자는 `admin` 역할에 속합니다.

{{site.data.alerts.callout_info}}
비밀번호 생성은 <code>root</code> 이외의 사용자에 대한 시큐어 클러스터에서만 지원됩니다. <code>root</code> 사용자는 클라이언트 인증서와 키로 인증해야 합니다.
{{site.data.alerts.end}}

### Logging

기본적으로 `user` 명령은 `stderr`에 오류를 기록합니다.

이 명령의 동작 문제를 해결하려면 [logging behavior](debug-and-error-logs.html)을 변경하십시오.

## 사용자 인증

시큐어 클러스터는 사용자가 데이터베이스와 테이블에 대한 액세스를 인증하도록 요구합니다. CockroachDB는 다음과 같은 두 가지 방법을 제공합니다:

- 모든 사용자가 사용할 수 있는 [클라이언트 인증서 및 키 인증](#secure-clusters-with-client-certificates). 최고 수준의 보안을 유지하려면 클라이언트 인증서와 키 인증만 사용하는 것이 좋습니다.

- 비밀번호를 생성받은 '루트'가 아닌 사용자가 사용할 수 있는 [암호 인증](#secure-clusters-with-passwords). `root`가 아닌 사용자를 위한 비밀번호를 설정하려면, `cockroach user set` 명령에 `--password` 플래그를 추가하십시오.

    사용자는 클라이언트 인증서와 키를 제공하지 않고도 비밀번호를 사용하여 인증할 수 있지만, 가능하면 인증서 기반 인증을 사용하는 것이 좋습니다.

    비밀번호 생성은 시큐어 클러스터에서만 지원됩니다.

## 예시

### 사용자 생성

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>
<p></p>

사용자 이름은 대소문자를 구분하지 않으며; 문자나 밑줄로 시작하고; 문자, 숫자 또는 밑줄만 포함해야 하며; 1에서 63자 사이여야 합니다.

<div class="filter-content" markdown="1" data-scope="secure">

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user set jpointsman --certs-dir=certs
~~~

{{site.data.alerts.callout_success}}사용자에 대해 암호 인증을 허용하려면 <code>--password</code> 플래그를 포함시킨 다음, 명령 프롬프트에 비밀번호를 입력하고 확인하십시오.{{site.data.alerts.end}}

사용자를 생성한 후 다음 작업을 수행하십시오:

- [클라이언트 인증서 생성](create-security-certificates.html#create-the-certificate-and-key-pair-for-a-client).
- [데이터베이스에 권한을 부여](grant.html).

</div>

<div class="filter-content" markdown="1" data-scope="insecure">

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user set jpointsman --insecure
~~~

사용자를 생성한 후에는 [데이터베이스에 권한을 부여](grant.html)해야 합니다.

</div>

### 특정(specific) 사용자로 인증

<div class="filters clearfix">
  <button style="width: 15%" class="filter-button" data-scope="secure">Secure</button>
  <button style="width: 15%" class="filter-button" data-scope="insecure">Insecure</button>
</div>
<p></p>

<div class="filter-content" markdown="1" data-scope="secure">

#### 클라이언트 인증서로 클러스터 보호

모든 사용자는 사용자 이름으로 발급된 [클라이언트 인증서](create-security-certificates.html#create-the-certificate-and-key-pair-for-a-client)를 사용하여 시큐어 클러스터에 대한 액세스를 인증할 수 있습니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=certs --user=jpointsman
~~~

#### 비밀번호로 클러스터 보호

비밀번호가 있는 사용자는 클라이언트 인증서와 키를 사용하는 대신 명령 프롬프트에서 비밀번호를 입력하여 액세스 권한을 인증할 수 있습니다.

사용자와 일치하는 클라이언트 인증서와 키 파일을 찾을 수 없으면 비밀번호 인증으로 대체됩니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --certs-dir=certs --user=jpointsman
~~~

</div>

<div class="filter-content" markdown="1" data-scope="insecure">

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure --user=jpointsman
~~~

</div>

### 사용자의 비밀번호 업데이트

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user set jpointsman --certs-dir=certs --password
~~~

이 명령을 실행한 후 명령 프롬프트에서 사용자의 새 비밀번호를 입력하고 확인하십시오.

비밀번호 생성은 `root`가 아닌 사용자에 대한 시큐어 클러스터에서만 지원됩니다. `root` 사용자는 클라이언트 인증서와 키로 인증해야 합니다.

### 모든 사용자 나열

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user ls --insecure
~~~

~~~
+------------+
|  username  |
+------------+
| jpointsman |
+------------+
~~~

### 특정 사용자 탐색

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user get jpointsman --insecure
~~~

~~~
+------------+--------------------------------------------------------------+
|  username  |                        hashedPassword                        |
+------------+--------------------------------------------------------------+
| jpointsman | $2a$108tm5lYjES9RSXSKtQFLhNO.e/ysTXCBIRe7XeTgBrR6ubXfp6dDczS |
+------------+--------------------------------------------------------------+
~~~

### 사용자 제거

{{site.data.alerts.callout_danger}}{% include {{ page.version.version }}/misc/remove-user-callout.html %}{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user rm jpointsman --insecure
~~~

{{site.data.alerts.callout_success}}또한 <a href="drop-user.html"><code>DROP USER</code></a> SQL 문을 사용하여 사용자를 제거할 수 있습니다.{{site.data.alerts.end}}

### 커맨드 라인 유틸리티에서 암시적으로 전송된 SQL 문 표시

이 예에서는 커맨드 라인 유틸리티에서 암시적으로 전송된 SQL 문을 표시하기 위해 `--echo-sql` 플래그를 사용합니다:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach user rm jpointsman --insecure --echo-sql
~~~

~~~
> DELETE FROM system.users WHERE username=$1
DELETE 1
~~~

## 더 보기

- [`CREATE USER`](create-user.html)
- [`DROP USER`](drop-user.html)
- [`SHOW USERS`](show-users.html)
- [`GRANT`](grant.html)
- [`SHOW GRANTS`](show-grants.html)
- [보안 인증서 생성](create-security-certificates.html)
- [역할 관리](roles.html)
- [기타 Cockroach 명령](cockroach-commands.html)
