# MySQL 한글 인코딩 문제 해결기 (Windows + Spring Boot)

## 문제 요약

- MySQL에서 한글이 `?덇툑`, `?낆텧湲?` 등으로 깨져서 출력됨
- `mysql: Character set 'utf8mb4' is not a compiled character set` 오류 발생
- front 에서 넘겨준 계좌생성시 계좌 유형 한글값이 정상적으로 dto를 통해 backend 까지 잘 넘어 오는것 확인
- 백엔드에서 db 저장시 에러가 없었고, 삽입은 되는 것은 확인, sql 을 통해 출력해보니 레코드 값의 한글이 깨진것을 확인
- 3시간 삽질 후 알게 된 내용에 대해 서술해봅니다..

---

## 원인 분석

| 항목 | 내용 |
|------|------|
| 구버전 MySQL (5.1) 사용 | utf8mb4 미지원 → 인코딩 깨짐 |
| 여러 버전 공존 | MySQL 5.1 (APM 설치) / 8.0.21 (WAMP 설치) |
| 환경 변수 PATH 오류 | 구버전 경로가 신버전보다 우선 |
| 문자셋 불일치 | 클라이언트 utf8 / 서버 utf8mb4 → 깨짐 발생 |

- 이런이유는, 데스크탑으로 프로젝트 진행중에 있어 mysql 버전 업그레이드 후 환경변수에 대한 설정값이 옛버전을 가리키고 있어
- cmd 에서는 예전 버전을 가리키고 있었고, 당연히 새로운 8버전 실행이 되지 않았음.
- 즉 환경변수 PATH 순서 문제.. -> MYSQL 5.1 구버전의 경로가 MYSQL 8.0 버전보다 우선순위에 존재하였음
- MYSQL 5.1 버전은 APM 설치 / MYSQL 8.0.21 버전은 WAMP 설치

- 문자셋의 불일치
- 클라이언트 : utf8
- 서버: utf8mb4
- 연결 문자셋의 불일치로 인한 데이터 깨짐도 경험해보았습니다.

- 설치되었던 mysql 관련 경로들
- C:\APM_Setup\Server\MySQL5\bin\mysql.exe (mysql 5.1)
- C:\Download\WAMP\mysql\bin\mysql.exe(mysql 8.0.21 실제로 운영중인 서버가 사용하는 mysql)
- C:\Program Files\MySQL\MySQL Workbench 8.0 CE\ (mysql workbench)

- SERVICE_NAME: APM_MYSQL5  (구버전 서비스)
- SERVICE_NAME: wampstackMySQL (신버전 서비스)
---

## 시도한 명령어

```bash
# 잘못된 클라이언트 실행 확인
mysql --version
→ mysql  Ver 14.14 Distrib 5.1.41

# 실제 실행 중인 서비스 경로 확인
sc qc wampstackMySQL
→ C:\Download\WAMP\mysql\bin\mysqld.exe

# MySQL 서비스 확인
sc query | findstr -i mysql

# 실행 중인 MySQL 프로세스 확인
wmic process where "name like '%mysql%'" get ExecutablePath


# 레지스트리 등록 경로 확인
reg query "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\wampstackMySQL" /v ImagePath

# 문자셋 확인
SHOW VARIABLES LIKE 'character%';

# 실행 중인 MySQL 프로세스 확인
wmic process where "name like '%mysql%'" get ExecutablePath, CommandLine

# 테이블 문자셋 확인
SHOW CREATE TABLE table_name;
---

## 해결과정
위 문제가 발생하였을때 먼저 확인했던 사항들을 정리합니다.
mysql 버전이 어느것인지?
클라이언트의 문자셋과 서버의 문자셋이 어떤것이며, 일치하는지?
환경변수 path 설정이 잘 되어있는지?
데이터베이스의 테이블 문자셋 확인하여 일치하는지
spring boot 연결 url 문자셋 설정이 잘되어 있는지 (useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC)

## 깨달은 점
업그레이드 및 다운그레이드, 또는 마이그레이션시 구버전 서비스가 살아있을 수 잇으므로 실행파일 충돌에 유의를 해야한다고 생각했습니다.
환경 변수 정리가 필수적인 작업이며, 문자셋을 서버 클라이언트 모두 통일해야 안정적인 데이터 삽입 및 조회를 할 수 있음을 깨달았습니다.

## 결론
이번 문제에서 겪었던 점은 이전의 설치하였던 구버전과 업데이트 한 버전간의 두 버전 충돌과 문자셋 불일치가 주된 원인이었습니다.
먼저 구버전이 신버전보다 앞서 실행이 되어 구버전 클라이언트가 실행되 utf8mb4를 지원하지 않아 한글이 깨지게 되었습니다.
spring boot의 jdbc 설정은 문제가 없다는것을 확인하였고 클라이언트와 서버의 문자셋 불일치로 인한 깨짐 발생을 이번 프로젝트를 진행하면서 몸소 느끼게 되었습니다.

명령어를 통하여 구버전과 신버전의 정확한 위치를 파악하였으며
my.ini 파일을 열어 문제가 없는지 검토하였습니다.

이후 적절한 문자셋 설정과 환경변수 재설정으로 인코딩 문제를 해결하였습니다.

