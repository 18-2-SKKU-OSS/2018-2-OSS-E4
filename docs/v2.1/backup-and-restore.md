---
title: Back up and Restore Data
summary: Learn how to back up and restore a CockroachDB database.
toc: true
redirect_from:
- back-up-data.html
- restore-data.html
---

CockroachDB는 장애에 대한 내성이 강하게 설계되었기 때문에 백업은 주로 재해 시 복구에만(즉, 클러스터가 대부분의 노드를 손실하는 경우) 필요하고, 격리된 문제(예: 소규모 노드 운영 중단)에는 개입이 필요하지 않습니다. 그러나 운영 모범 사례로서는, 데이터를 정기적으로 백업하는 것을 권장합니다.

[라이선스 유형](https://www.cockroachlabs.com/pricing/)에 따라, CockroachDB는 클러스터의 데이터를 백업 및 복원하는 두 가지 방법을 제안합니다: 엔터프라이즈와 코어

## 엔터프라이즈 백업 및 복원

[엔터프라이즈 라이선스](enterprise-licensing.html)가 있는 경우, [`BACKUP`](backup.html) 문을 사용하여 클러스터의 스키마와 데이터를 AWS S3, Google 클라우드 스토리지 또는 NFS와 같은 인기 클라우드 서비스에 효율적으로 백업하고,  [`RESTORE`](restore.html) 문을 사용하여 스키마를 효율적으로 복원할 수 있습니다.

### 수동 전체 백업

대부분의 경우 클러스터에 있는 각 데이터베이스를 전체 야간 백업하려면 [`BACKUP`](backup.html) 명령을 사용하는 것을 권장합니다.

{% include copy-clipboard.html %}
~~~ sql
> BACKUP DATABASE <database_name> TO '<full_backup_location>';
~~~

If it's ever necessary, you can then use the [`RESTORE`](restore.html) command to restore a database:

{% include copy-clipboard.html %}
~~~ sql
> RESTORE DATABASE <database_name> FROM '<full_backup_location>';
~~~

### 수동 전체 백업 및 증분 백업

데이터베이스가 야간 전체 백업을 수행할 수 없는 크기로 증가하면, 야간 증분 백업과 함께 정기적인 전체 백업(예: 매주)을 수행하는 것을 고려해 보십시오. 증분 백업은 대용량 데이터베이스의 전체 백업보다 빠르고 스토리지 효율적입니다.

[`BACKUP`](backup.html)명령을 주기적으로 실행하여 데이터베이스 전체 백업:

{% include copy-clipboard.html %}
~~~ sql
> BACKUP DATABASE <database_name> TO '<full_backup_location>';
~~~

그런 다음 이미 생성한 전체 백업을 기반으로 야간 증분 백업을 생성하십시오.

{% include copy-clipboard.html %}
~~~ sql
> BACKUP DATABASE <database_name> TO 'incremental_backup_location'
INCREMENTAL FROM '<full_backup_location>', '<list_of_previous_incremental_backup_location>';
~~~

필요할 경우 [`RESTORE`](restore.html) 명령을 사용하여 데이터베이스를 복원하십시오.

{% include copy-clipboard.html %}
~~~ sql
> RESTORE <database_name> FROM '<full_backup_location>', '<list_of_previous_incremental_backup_locations>';
~~~

### 자동 전체 백업 및 증분 백업

스크립트를 사용하거나 cron job 등 원하는 자동화 방법을 사용하여 백업을 자동화할 수 있습니다.

백업을 자동화하도록 사용자 정의할 수 있도록 작성한 [샘플 백업 스크립트](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/prod-deployment/backup.sh)를 참조하십시오.

샘플 스크립트에서 전체 백업을 생성할 요일을 지정하십시오. 매일 스크립트를 실행하면 지정된 날짜에 전체 백업이 생성되며, 다른 날에는 증분 백업이 생성됩니다. 이 스크립트는 최근 생성된 백업을 `backup.txt`라는 별도의 파일에서 추적하며 이 파일을 후속 증분 백업의 기반으로 사용합니다.

1. [샘플 백업 스크립트](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/prod-deployment/backup.sh) 다운로드:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ wget -qO- https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/{{ page.version.version }}/prod-deployment/backup.sh
    ~~~

    Alternatively, you can create the file yourself and copy the script into it:

    {% include copy-clipboard.html %}
    ~~~ shell
    #!/bin/bash

    set -euo pipefail

    # This script creates full backups when run on the configured
    # day of the week and incremental backups when run on other days, and tracks
    # recently created backups in a file to pass as the base for incremental backups.

    full_day="<day_of_the_week>"                      # Must match (including case) the output of `LC_ALL=C date +%A`.
    what="DATABASE <database_name>"                   # The name of the database you want to back up.
    base="<storage_URL>/backups"                      # The URL where you want to store the backup.
    extra="<storage_parameters>"                      # Any additional parameters that need to be appended to the BACKUP URI (e.g., AWS key params).
    recent=recent_backups.txt                         # File in which recent backups are tracked.
    backup_parameters=<additional backup parameters>  # e.g., "WITH revision_history"

    # Customize the `cockroach sql` command with `--host`, `--certs-dir` or `--insecure`, and additional flags as needed to connect to the SQL client.
    runsql() { cockroach sql --insecure -e "$1"; }

    destination="${base}/$(date +"%Y%m%d-%H%M")${extra}"

    prev=
    while read -r line; do
        [[ "$prev" ]] && prev+=", "
        prev+="'$line'"
    done < "$recent"

    if [[ "$(LC_ALL=C date +%A)" = "$full_day" || ! "$prev" ]]; then
        runsql "BACKUP $what TO '$destination' AS OF SYSTEM TIME '-1m' $backup_parameters"
        echo "$destination" > "$recent"
    else
        destination="${base}/$(date +"%Y%m%d-%H%M")-inc${extra}"
        runsql "BACKUP $what TO '$destination' AS OF SYSTEM TIME '-1m' INCREMENTAL FROM $prev $backup_parameters"
        echo "$destination" >> "$recent"
    fi

    echo "backed up to ${destination}"
    ~~~

2. 샘플 백업 스크립트에서 다음 변수에 대한 값을 사용자 정의하십시오:

    Variable | Description
    -----|------------
    `full_day` | 전체 백업을 하고 싶은 요일.
    `what` | 백업할 데이터베이스의 이름(즉, 데이터베이스에 모든 테이블과 보기의 백업 생성).
    `base` | 백업을 저장할 URL.<br/><br/>URL 포맷: `[scheme]://[host]/[path]` <br/><br/> URL의 구성 요소에 대한 자세한 내용은 [백업 파일 URL](backup.html#backup-file-urls)을 참조하십시오.
    `extra`| 스토리지에 필요한 매개 변수.<br/><br/>매개 변수 형식: `?[parameters]` <br/><br/> 스토리지 매개 변수에 대한 자세한 내용은 [백업 파일 URL](backup.html#backup-file-urls을 참조하십시오.
    `backup_parameters` | 추가로 지정하고자 하는 [백업 매개 변수](backup.html#parameters).

    또한 `--host`, `--certs-dir` 또는 `--insecure`, 그리고 필요에 따라 [추가 플래그](use-the-built-in-sql-client.html#flags)를 사용하여 `cockroach sql` 명령을 사용자 정의하십시오.

3. 스크립트를 실행할 수 있도록 파일 권한 변경:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ chmod +x backup.sh
    ~~~

4. 백업 스크립트 실행:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ./backup.sh
    ~~~

{{site.data.alerts.callout_info}}
증분 백업이 누락된 경우  `recent_backups.txt` 파일을 삭제하고 스크립트를 실행하십시오. 이는 그날 전체 백업 및 이후 날짜의 증분 백업을 가져옵니다.
{{site.data.alerts.end}}

## 코어 백업 및 복구

엔터프라이즈 라이선스가 없는 경우 코어 백업을 수행할 수 있습니다. [`cockroach dump`](sql-dump.html) 명령을 실행하여 데이터베이스의 모든 테이블을 새 파일(다음 예의 `backup.sql`)로 덤프하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach dump <database_name> <flags> > backup.sql
~~~

코어 백업에서 데이터베이스를 복원하려면 [`cockroach sql` 명령을 사용하여 백업 파일에서 명령 실행](sql-dump.html#restore-a-table-from-a-backup-file)을 참조하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --database=[database name] < backup.sql
~~~

{{site.data.alerts.callout_success}}
다른 데이터베이스에서 백업을 만들어 CockroachDB로 가져오려면 [데이터 가져오기](import-data.html)를 참조하십시오.
{{site.data.alerts.end}}

## 더 알아보기

- [`BACKUP`](backup.html)
- [`RESTORE`](restore.html)
- [`SQL DUMP`](sql-dump.html)
- [`IMPORT`](import-data.html)
- [빌트인 SQL 클라이언트 사용](use-the-built-in-sql-client.html)
- [다른 Cockroach 명령어](cockroach-commands.html)
