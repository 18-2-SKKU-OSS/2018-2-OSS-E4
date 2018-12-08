---
title: Manage Roles
summary: Roles are SQL groups that contain any number of users and roles as members.
toc: true
---

역할은 구성원으로 여러 사용자와 역할을 포함하는 SQL 그룹입니다. 클러스터 역할을 생성하고 관리하려면, 다음 명령문을 사용하십시오:

- [`CREATE ROLE` (엔터프라이즈)](create-role.html)
- [`DROP ROLE` (엔터프라이즈)](drop-role.html)
- [`GRANT <roles>` (엔터프라이즈)](grant-roles.html)
- [`REVOKE <roles>` (엔터프라이즈)](revoke-roles.html)
- [`GRANT <privileges>`](grant.html)
- [`REVOKE <privileges>`](revoke.html)
- [`SHOW ROLES`](show-roles.html)
- [`SHOW GRANTS`](show-grants.html)


## 용어

시작하기 위해, 기본 역할 용어가 아래에 요약되어 있습니다:

용어 | 설명
-----|------------
역할 | 임의의 수의 [사용자](create-and-manage-users.html) 또는 다른 역할을 포함하는 그룹.<br><br>참고: 모든 사용자는 권한을 [부여](grant.html)하고 권한을 [취소](revoke.html)할 수 있는 `public`역할을 담당합니다.
역할 관리자 | 역할 구성원을 수정할 수 있는 역할의 구성원. 역할 관리자를 생성하려면, [`WITH ADMIN OPTION`](grant-roles.html#grant-the-admin-option)을 사용하십시오.
수퍼 유저 / 관리자 | `admin` 역할의 구성원. 수퍼 유저만 [`CREATE ROLE`](create-role.html) 또는 [`DROP ROLE`](drop-role.html)할 수 있습니다. `admin` 역할은 기본적으로 생성되며 삭제될 수 없습니다.
`root` | 기본적으로 `admin` 역할의 구성원으로 존재하는 사용자. `root` 사용자는 항상 `admin` 역할의 구성원이어야 합니다.
상속 | 구성원에게 역할의 권한을 부여하는 동작.
다이렉트 구성원 | 역할의 직접적인 구성원인 사용자 또는 역할.<br><br>예: `A`는 `B`의 구성원입니다.
인다이렉트 구성원 | 연결별 역할의 구성원인 사용자 또는 역할.<br><br> `A`는 `C`의 구성원입니다 ...는 "..."가 임의의 회원 수인 `B`의 구성원입니다.  

## 예제

이 예제의 경우, 인시큐어 모드에서 실행되는 [엔터프라이즈 라이센스](enterprise-licensing.html)와 하나의 CockroachDB 노드가 필요합니다.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--store=roles \
--listen-addr=localhost:26257
~~~

1. `root` 사용자로서,[`cockroach user`](create-and-manage-users.html) 명령을 사용하여 새로운 사용자인 `maxroach`를 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach user set maxroach --insecure
    ~~~

2. `root`사용자로서, [빌트인 SQL 클라이언트](use-the-built-in-sql-client.html)를 여십시오:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

3. 데이터베이스를 생성하고 그것을 기본값으로 설정하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE test_roles;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET DATABASE = test_roles;
    ~~~

4. [역할을 생성](create-role.html)하고 다음 데이터베이스에 [모든 역할을 나열](show-roles.html)하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE ROLE system_ops;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW ROLES;
    ~~~

    ~~~
    +------------+
    |  rolename  |
    +------------+
    | admin      |
    | system_ops |
    +------------+
    ~~~

5. 생성한 `system_ops` 역할에 권한을 부여하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > GRANT CREATE, SELECT ON DATABASE test_roles TO system_ops;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW GRANTS ON DATABASE test_roles;
    ~~~

    ~~~
    +------------+--------------------+------------+------------+
    |  Database  |       Schema       |    User    | Privileges |
    +------------+--------------------+------------+------------+
    | test_roles | crdb_internal      | admin      | ALL        |
    | test_roles | crdb_internal      | root       | ALL        |
    | test_roles | crdb_internal      | system_ops | CREATE     |
    | test_roles | crdb_internal      | system_ops | SELECT     |
    | test_roles | information_schema | admin      | ALL        |
    | test_roles | information_schema | root       | ALL        |
    | test_roles | information_schema | system_ops | CREATE     |
    | test_roles | information_schema | system_ops | SELECT     |
    | test_roles | pg_catalog         | admin      | ALL        |
    | test_roles | pg_catalog         | root       | ALL        |
    | test_roles | pg_catalog         | system_ops | CREATE     |
    | test_roles | pg_catalog         | system_ops | SELECT     |
    | test_roles | public             | admin      | ALL        |
    | test_roles | public             | root       | ALL        |
    | test_roles | public             | system_ops | CREATE     |
    | test_roles | public             | system_ops | SELECT     |
    +------------+--------------------+------------+------------+
    ~~~

6. `maxroach` 사용자를 `system_ops` 역할에 추가하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > GRANT system_ops TO maxroach;
    ~~~

7. `system_ops` 역할에 방금 추가한 권한을 테스트하려면, `\q` 또는 `ctrl-d`를 사용하여 대화형 쉘을 종료한 다음, 쉘을 다시 `maxroach` 사용자로 엽니다 (누가 `system_ops` 역할의 구성원인지):

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --user=maxroach --database=test_roles --insecure
    ~~~

8. `maxroach` 사용자로서, 테이블을 생성하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE employees (
        id UUID DEFAULT uuid_v4()::UUID PRIMARY KEY,
        profile JSONB
      );
    ~~~

    `maxroach`는 `CREATE` 권한을 가지고 있기 때문에 우리는 테이블을 생성할 수 있었습니다.

9. `maxroach` 사용자로서, 테이블을 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > DROP TABLE employees;
    ~~~

    ~~~
    pq: user maxroach does not have DROP privilege on relation employees
    ~~~

    현재 사용자 (`maxroach`)가 `DROP` 권한이 없는 `system_ops` 역할의 멤버이기 때문에, 테이블을 삭제할 수 없습니다.

10. `maxroach` 는 `CREATE`와`SELECT` 권한을 가지므로, `SHOW` 명령문을 사용하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW GRANTS ON TABLE employees;
    ~~~

    ~~~
    +------------+--------+-----------+------------+------------+
    |  Database  | Schema |   Table   |    User    | Privileges |
    +------------+--------+-----------+------------+------------+
    | test_roles | public | employees | admin      | ALL        |
    | test_roles | public | employees | root       | ALL        |
    | test_roles | public | employees | system_ops | CREATE     |
    | test_roles | public | employees | system_ops | SELECT     |
    +------------+--------+-----------+------------+------------+
    ~~~

11. 이제 역할과 관련된 더 많은 SQL 명령문을 테스트하기 위해 `root` 사용자로 다시 전환하십시오. 대화식 쉘을 종료하려면, `\q` 또는 `ctrl-d`를 사용하고, 쉘을 `root` 사용자로 다시 엽니다.

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure
    ~~~

12. `root` 사용자로서 권한을 취소한 다음 `system_ops` 역할을 삭제하십시오:

    {% include copy-clipboard.html %}
    ~~~ sql
    > REVOKE ALL ON DATABASE test_roles FROM system_ops;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW GRANTS ON DATABASE test_roles;
    ~~~
    ~~~
    +------------+--------------------+-------+------------+
    |  Database  |       Schema       | User  | Privileges |
    +------------+--------------------+-------+------------+
    | test_roles | crdb_internal      | admin | ALL        |
    | test_roles | crdb_internal      | root  | ALL        |
    | test_roles | information_schema | admin | ALL        |
    | test_roles | information_schema | root  | ALL        |
    | test_roles | pg_catalog         | admin | ALL        |
    | test_roles | pg_catalog         | root  | ALL        |
    | test_roles | public             | admin | ALL        |
    | test_roles | public             | root  | ALL        |
    +------------+--------------------+-------+------------+
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > REVOKE ALL ON TABLE test_roles.* FROM system_ops;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW GRANTS ON TABLE test_roles.*;
    ~~~
    ~~~
    +------------+--------+-----------+-------+------------+
    |  Database  | Schema |   Table   | User  | Privileges |
    +------------+--------+-----------+-------+------------+
    | test_roles | public | employees | admin | ALL        |
    | test_roles | public | employees | root  | ALL        |
    +------------+--------+-----------+-------+------------+
    ~~~

    {{site.data.alerts.callout_info}}역할이나 사용자의 권한은 모두 삭제되기 전에 취소되어야 합니다.{{site.data.alerts.end}}

    {% include copy-clipboard.html %}
    ~~~ sql
    > DROP ROLE system_ops;
    ~~~

## 더 보기

- [`CREATE ROLE`](create-role.html)
- [`DROP ROLE`](drop-role.html)
- [`SHOW ROLES`](show-roles.html)
- [`GRANT <privileges>`](grant.html)
- [`GRANT <roles>` (Enterprise)](grant-roles.html)
- [`REVOKE <privileges>`](revoke.html)
- [`REVOKE <roles>` (Enterprise)](revoke-roles.html)
- [`SHOW GRANTS`](show-grants.html)
- [사용자 관리](create-and-manage-users.html)
- [권한](privileges.html)
- [다른 Cockroach 명령어들](cockroach-commands.html)
