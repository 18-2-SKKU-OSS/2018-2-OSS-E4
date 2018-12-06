---
title: Enterprise Licensing
summary: Request and set trial and enterprise license keys for CockroachDB
toc: true
---

CockroachDB는 핵심적인 [엔터프라이즈 기능](https://www.cockroachlabs.com/pricing/)을 모두 포함하는 단일 바이너리를 배포합니다. 라이센스 키 없이 핵심 기능을 사용할 수 있습니다. 그러나 엔터프라이즈 기능을 사용하려면, 평가판 또는 엔터프라이즈 라이센스 키가 필요합니다.

이 페이지에서는 CockroachDB에 대한 평가판 및 엔터프라이즈 라이센스 키를 얻고 설정하는 방법을 보여줍니다.


## 라이센스 타입

타입 | 설명
-------------|------------
**평가판 라이센스** | 평가판 라이센스를 통해 CockroachDB 엔터프라이즈 기능을 30일 동안 무료로 사용해 볼 수 있습니다.
**엔터프라이즈 라이센스** | 유료 엔터프라이즈 라이센스를 사용하면 CockroachDB 엔터프라이즈 기능을 장기간 (1년 이상) 사용할 수 있습니다.

## 평가판 또는 엔터프라이즈 라이센스 키 얻기

평가판 라이센스 키를 얻으려면, [등록 양식](https://www.cockroachlabs.com/get-cockroachdb/)을 작성하고 몇 분 내에 평가판 라이센스 키를 이메일로 받으십시오.

엔터프라이즈 라이센스로 업그레이드하려면, <a href="mailto:sales@cockroachlabs.com"> 판매 팀에 문의</a>하십시오.

## 평가판 또는 엔터프라이즈 라이센스 키 설정

CockroachDB `root` 사용자로서, CockroachDB 설정에 따라, 인시큐어 혹은 시큐어 모드에서 [빌트인 SQL 쉘](use-the-built-in-sql-client.html)을 여십시오. 다음 예시에서는, CockroachDB가 인시큐어 모드로 실행되고 있다고 가정합니다. 그런 다음, `SET CLUSTER SETTING` 명령을 사용하여 조직의 이름과 라이센스 키를 설정하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SET CLUSTER SETTING cluster.organization = 'Acme Company';
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SET CLUSTER SETTING enterprise.license = 'xxxxxxxxxxxx';
~~~

## 라이센스 키 확인

라이센스 키를 확인하려면, [빌트인 SQL 쉘](use-the-built-in-sql-client.html)을 열고 `SHOW CLUSTER SETTING` 명령을 사용하여 조직 이름과 라이센스 키를 확인하십시오:

{% include copy-clipboard.html %}
~~~ sql
>  SHOW CLUSTER SETTING cluster.organization;
~~~
~~~
+----------------------+
| cluster.organization |
+----------------------+
| Acme Company         |
+----------------------+
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SHOW CLUSTER SETTING enterprise.license;
~~~
~~~
+--------------------------------------------------------------------+
|                         enterprise.license                         |
+--------------------------------------------------------------------+
| xxxxxxxxxxxx                                                       |
+--------------------------------------------------------------------+
(1 row)
~~~

라이센스 설정은 명령이 실행되는 노드의 cockroach.log에도 기록됩니다.

{% include copy-clipboard.html %}
~~~ sql
$ cat cockroach.log | grep license
~~~
~~~
I171116 18:11:48.279604 1514 sql/event_log.go:102  [client=[::1]:56357,user=root,n1] Event: "set_cluster_setting", target: 0, info: {SettingName:enterprise.license Value:xxxxxxxxxxxx User:root}
~~~
 
## 만료된 라이센스 갱신

라이센스가 만료되면, 엔터프라이즈 기능이 작동하지 않지만, 프로덕션 설정에는 영향을 미치지 않습니다. 예를 들어, 라이센스가 갱신될 때까지 백업 및 복원 기능이 작동하지 않지만, CockroachDB의 다른 모든 기능을 중단없이 계속 사용할 수 있습니다.

만료된 라이센스를 갱신하려면, <a href="mailto:sales@cockroachlabs.com">판매팀에 문의</a>하거나 새로운 라이센스를 [설정](enterprise-licensing.html#set-the-trial-or-enterprise-license-key)하십시오. 

## 더 보기

- [`클러스터 설정`](set-cluster-setting.html)
- [`클러스터 설정 표시`](show-cluster-setting.html)
- [엔터프라이즈 평가판 –– 시작](get-started-with-enterprise-trial.html)
