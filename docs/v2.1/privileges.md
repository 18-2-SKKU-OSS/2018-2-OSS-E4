---
title: Privileges
summary: Privileges are granted to roles and users at the database and table levels. They are not yet supported for other granularities such as columns or rows.
toc: true
---

CockroachDB에서, 데이터베이스 및 테이블 레벨에서 [역할](roles.html) 및 [사용자](create-and-manage-users.html)에 권한이 부여됩니다. 그것들은 열이나 행과 같은 기타 세분화에 대해서는 아직 지원되지 않다.

사용자가 [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html) 또는 [클라이언트 드라이버](install-client-drivers.html)를 통해 데이터베이스에 연결하면, CockroachDB는 실행된 각 명령문에 대한 사용자 및 역할의 권한을 확인합니다. 사용자에게 명령문에 대한 충분한 권한이 없는 경우, CockroachDB는 오류를 발생시킵니다.

특정 명령문에 필요한 특권에 대해서는, 각각의 [SQL 명령문](sql-statements.html)에 대한 설명서를 참고하십시오. 


## 지원되는 권한

지원되는 전체 권한 목록은 [`GRANT`](grant.html) 설명서를 보십시오.

## 권한 부여

역할이나 사용자에게 권한을 부여하려면, [`GRANT`](grant.html) 명령문을 사용하십시오. 예를 들면:

{% include copy-clipboard.html %}
~~~ sql
> GRANT SELECT, INSERT ON bank.accounts TO maxroach;
~~~

## 권한 표시

역할이나 사용자에게 부여된 권한을 표시하려면, [`SHOW GRANTS`](show-grants.html) 명령문을 사용하십시오. 예를 들면:

{% include copy-clipboard.html %}
~~~ sql
> SHOW GRANTS ON DATABASE bank FOR maxroach;
~~~

## 권한 취소

역할이나 사용자로부터 권한을 취소하려면, [`REVOKE`](revoke.html) 명령문을 사용하십시오. 예를 들면:

{% include copy-clipboard.html %}
~~~ sql
> REVOKE INSERT ON bank.accounts FROM maxroach;
~~~

## 더 보기

- [사용자 관리](create-and-manage-users.html)
- [역할 관리](roles.html)
- [SQL 명령문](sql-statements.html)
