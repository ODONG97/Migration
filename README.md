# Migration

# Check List - 물리
- DB Object 구조
- Migration을 위한 가용 Disk Size
- OS 제한 File Size 존재 유무 (File Syste:32GB)
- Distribute Database Option의 Operation 가능 유무
- Down Time을 줄이며 작업 효율을 향상 시킬 수 있는 Resource 정보
- CPU, Memory, Network Speed, Disk I/O
- 고객사 보안 정책 문제 (DB 암호화, 접근 제어, 감사)
- 기타 물리적 제약 유무

# Check List - 논리
- Migration시 허용 된 Down Time 시간
- 향후 Storage 운영 계획(통/폐합에 따른 Storage 변경, 성능 및 가용성을 고려한 Disk 배치(Raid), Data 증가에 따른 Size 확보 or Data Purge 정책 or Compression)
- 기존 업무 System 과 Interface 이상 유무 점검
- Oracle 12c New Feature 적용 유무
- Migration 으로 인한 Down Time 발생 시 OLTP 운영 방안
- Migration 후 Data 정합성 검증
- Migration 후 Application 이상 유무 검증
- 성능 점검

# Check List - 방법
- Oracle Data Pump ( + export / import )
- Oracle Transportable Tablespaces
- SQL*Loader
- External Tables
- CTAS & DB_LINK

# General Architecture
- DBMS_DATAPUMP ( 대량의 데이터 / 메타 데이터 이동을 위한 고속 export,import Utilty )
- Direct Path API(DPAPI) ( 데이터 변환 및 구문 분석을 최소화하는 Direct Path API Interface 제공 )
- DBMS_METADATA ( 모든 메타 데이터의 Unloading/Loading을 위해 Worker 프로세스에서 사용, Database Object 정의는 SQL 대신 XML로 저장 )
- External Table API ( ORACLE_DATAPUMP 및 ORACLE_LOADER Access Driver를 사용하여 External Table에 데이터를 저장 )
- SQL*Loader ( External Table과 통합되어 External Table Access Parameter에 Loader Control File에 자동 이전 기능 제공 )
- Expdp and Impdp ( Data Pump 작업을 시작 및 모니터하기 위해 DBMS_DATAPUMP 패키지를 호출하는 Thin Layer )
- Other Clients ( 이 Infrastructure를 활용하는 응용 프로그램 -> Replication, Transportable, Tablespaces )

# Oracle Data Pump (+ export / import)
- 기존의 Export / Import Utility 는 여러 플랫폼에서 데이터를 전송하기 위해 사용
- Import 수행 시 단순히 Export 된 Dump 파일에서 각 레코드를 읽고 이를 일반적인 Insert Into 명령어를 사용해서 대상 테이블에 삽입 
- XML 형식으로 파일을 생성
   ㄴ 기존 SQL문을 사용했던것보다 정교한 필터링과 이행 성능, 속도 개선
- Oracle 10g 에서 일반적인 SQL 문을 사용하는 대신 독점 API 로 데이터를 빠르게 로드 및 언로드
- Direct 모드의 Export 보다 10 ~ 15 배 향상 , Import 프로세스 성능도 5 배 향상
- 정교한 필터링 & 이행 성능

* Data Pump Key Features *
- Fast Performance
 ㄴ 기존 Export and Import 유틸리티보다 훨씬 빠르다. Import에서 단순히 export dump 파일에서 레코드를 읽고 일반적인 insert into 명령을 사용하여 대상 테이블에 삽입 하는 대신 Data Pump Import는 Direct Path Method Loading을 사용하기 때문
 - Imporved Management Restart
  ㄴ 모든 Data Pump operation 은 Data Pump job 을 실행하는 스키마에 만들어진 master table 을 가지고 있다. Master table 은 현재 수행중인 모든 export 또는 import 시 객체의 상태정보와 dump file set 에서의 위치정보를 가지고 있다. 이는 갑작스런 job 의 중단에도 job 의 성공적인 종료에 상관 없이 어떤 object 의 작업이 진행 중이었는지 알 수 있게 해 준다. 그래서 master table 과 dump file set 이 있는 한 모든 정지된 data pump job 은 데이터 손실 없이 다시 시작할 수 있다. 
- Fine-Grained Object Selection
 ㄴ Data Pump Job은 거의 모든 Type의 Object를 exclude 또는 include 시킬 수 있다.
 @ EXCLUDE - 특정 객체 유형을 제외한다. 
 @ INCLUDE - 특정 객체 유형을 포함한다.
 @ CONTENT - 로드를 취소 할 데이터를 지정한다.
 @ QUERY - 테이블 부분 집합을 EXPORT 하기 위해 사용
 - Monitoring and Estimating Capability
  ㄴ Data Pump는 Standard Progress, Error Message를 Log File에 기록 할 뿐 아니라 현재 Operation 상태를 대화식모드로 보여준다.
 - Network Mode
  ㄴ Data Pump Export and Import는 job의 source가 리모트 인스턴스일 경우 network mode를 지원한다.
  
# 사용 전 환경 설정하기
1. Data Pump 는 Export/Import 와 다르게 유틸리티가 직접 OS 파일에 I/O 를 할 수 없고 오라클 Directory 라는 객체를 통해서 간접으로 접근 가능 
2. Data Pump 를 사용하기 위해서는 미리 Directory가 만들어져 있어야 하며 Data Pump를 수행하는 사용자는 그 Directory에 접근할 수 있는 권한이 있어야 함
3. 이 기능을 통해 DBA는 Data Pump 의 보안 관리까지 가능

Oracle Data Pump
[root] mkdir -p /data/datapump   => datapump 디렉토리 생성
[root] chown -R oracle.oinstall /data/   
[root] chmod -R 755 /data/

SYS@.. > create or replace directory datapump as '/data/datapump';
-> Directory created.
SYS@.. > grant read, write on directory datapump to scott;
-> Grant succeeded.
SYS@.. > select * from dba_directories;

# Oracle Data Pump - Expdp 실행 모드
1. Database
- Full 파라미터를 사용하며 데이터베이스 전체를 Export
- DBA 권한을 가지고 있거나 Export_Full_Database 권한을 가지고 있어야 수행 가능.
2. Schemas
- Schemas 파라미터를 사용하며 특정 스키마 전체를 Export
3. Table
- Tables 파라미터를 사용하여 여러 개의 테이블을 Export 받으려면 , 로 구분

