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

# Oracle Data Pump (+ export / import) : Export,Import를 대체하기 위해 Oracle Database 10g 이후 버전에서 제공되는 Utility
- 기존의 Export / Import Utility 는 여러 플랫폼에서 데이터를 전송하기 위해 사용, Original Export/Import와 유사한 동작을 하지만 Data Pump가 효율적으로 동작
- Import 수행 시 단순히 Export 된 Dump 파일에서 각 레코드를 읽고 이를 일반적인 Insert Into 명령어를 사용해서 대상 테이블에 삽입 
- XML 형식으로 파일을 생성
   ㄴ 기존 SQL문을 사용했던것보다 정교한 필터링과 이행 성능, 속도 개선
- Oracle 10g 에서 일반적인 SQL 문을 사용하는 대신 독점 API 로 데이터를 빠르게 로드 및 언로드
- Direct 모드의 Export 보다 10 ~ 15 배 향상 , Import 프로세스 성능도 5 배 향상
- 정교한 필터링 & 이행 성능
- Data Pump Export는 데이터베이스에 저장되어 있는 데이터들을 OS의 바이너리 파일로 전환하는 도구
- Data Pump Import는 바이너리 파일을 데이터베이스 안의 데이터로 전환하는 도구

* Data Pump Key Features *
- Fast Performance
 ㄴ 기존 Export and Import 유틸리티보다 훨씬 빠르다(약 2배-Direct Path API가 효과적으로 수정). Import에서 단순히 export dump 파일에서 레코드를 읽고 일반적인 insert into 명령을 사용하여 대상 테이블에 삽입 하는 대신 Data Pump Import는 Direct Path Method Loading을 사용하기 때문(Parallelism의 Level에 따라서 더욱 향상된 Performance를 보여줌)
 - Imporved Management Restart
  ㄴ 모든 Data Pump operation 은 Data Pump job 을 실행하는 스키마에 만들어진 master table 을 가지고 있다. Master table 은 현재 수행중인 모든 export 또는 import 시 객체의 상태정보와 dump file set 에서의 위치정보를 가지고 있다. 이는 갑작스런 job 의 중단에도 job 의 성공적인 종료에 상관 없이 어떤 object 의 작업이 진행 중이었는지 알 수 있게 해 준다. 그래서 master table 과 dump file set 이 있는 한 모든 정지된 data pump job 은 데이터 손실 없이 다시 시작할 수 있다. 
- Fine-Grained Object Selection
 ㄴ Data Pump Job은 거의 모든 Type의 Object를 exclude 또는 include 시킬 수 있다.
 @ EXCLUDE - 특정 객체 유형을 제외한다. 
 @ INCLUDE - 특정 객체 유형을 포함한다.
 @ CONTENT - 로드를 취소 할 데이터를 지정한다.
 @ QUERY - 테이블 부분 집합을 EXPORT 하기 위해 사용
 - Monitoring and Estimating Capability
  ㄴ Data Pump는 Standard Progress, Error Message를 Log File에 기록 할 뿐 아니라 현재 Operation 상태를 대화식모드로 보여준다. Job 의 completion percentage 를 측정하여 보여주며 초 단위의 지정한 time period 에 따라 자동으로 update 하여 표시한다. 1 개 이상의 client 가 running job 에 attach 할 수 있기 때문에 업무환경에서 job 을 실행하고, detach 한 후 집에 가서 job 을 reattach 하여 끊김 없이 모든 job 을 모니터링 할 수 있다. 모든export job이 시작할 때 대략적인 전체unload 양을 측정해 준다. 이는 사용자가 dump file set을 위한 충분한 양의 disk space를 할당할 수 있게 한다. 
 - Network Mode
  ㄴ Data Pump Export and Import는 job의 source가 리모트 인스턴스일 경우 network mode를 지원한다. Network을 통해 import를 할 때 source가 dump file set이 아닌 다른 database에 있기 때문에 dump file이 없다. Network 를 통해 export 를 할 때 souce 가 다른 시스템에 있는 read-only database 일 수 있다. Dumpfile 은 local(non-networked)export 처럼 local 시스템에 쓰이게 된다. 
  
# Oracle Data Pump 개요
Master Table, Master Process, Worker Process를 사용하여 작업을 수행 및 진행 상황을 추적
● Coordination of a job : 모든 Data Pump Export 및 Data Pump Import 작업을 조정하기 위한 Master Process가 생성됩니다. Client와의 통신, Worker Process 생성 및 제어, Logging 작업 수행을 포함
SQL > select to_char (sysdate, 'YYYY-MM-DD HH24:Ml:SS') "DATE", s.program, s.sid, s.status, s.username, d.job_name, p.spid, s.serial#, p.pid 
      from v$session s, v$process p, dba_datapump_sessions d
      where p.addr=s.paddr and s.saddr=d.saddr
      
● Tracking Progress Within a Job : Data와 Meta Data가 전송되는 동안 Master Table은 작업 내에서 진행 상황을 추적하는데 사용 / Export 또는 Import 조작을 수행하는 현재 스키마에 테이블로 구현 / Master Table의 존재로 인해 Data Pump 작업의 중지 및 재시작이 가능 / 수행하는 스키마에 Create Table 권핞 및 Master Table을 생성하기 위한 충분한 Tablespace 할당량 필요
-> Export 작업
      Master Table은 Dump File Set에 있는 Database Object의 위치 기록
      작업 동안 Master Table을 빌드하고 유지 관리
      작업이 끝나면 Master Table의 내용을 Dump File Set에 기록
-> Import 작업
      Master Table은 Dump File Set에서 Load 되며 Target Database로 Import 해야하는 Object를 찾기 위한 조작 순서 제어하는데 사용

● Filtering Data and Metadata During a Job : EXCLUDE&INCLUDE 매개 변수를 사용하여 EXPORT&IMPORT 된 Object 유형 필터 가능 / Master Table에 특정 Object는 이름 또는 소유 스키마와 같은 속성 명시 / Object는 Object Class(Table, Index, Directory)에 속함 / Object 유형을 제한 가능 / Data 특정 필터를 지정하여 행도 제한 가능
   [Master Table]
      Select BASE_OBJECT_NAME, BASE_OBJECT_SCHEMA, COMPLETED_ROWS, COMPLETION_TIME
      FROM HR.SYS_EXPORT_SCHEMA_06
      WHERE PROCESS_ORDER > 0
      AND DUPLICATE = 0
      AND OBJECT_TYPE = 'TABLE_DATA';

● Transforming Metadata During a Job : Data Pump Import 매개 변수 Remap_Datafile, Remap_Schema, Remap_Table, Remap_Tablespace, Transform 및 Partition_Options을 사용하여 Meta Data의 변환을 수행할 수 있다.

● Maximizing Job Performance : Data Pump는 병렬로 실행되는 여러 작업자 Process를 사용하여 작업 성능을 향상시킬 수 있다. / PARALLEL Parameter 사용 / 병렬 처리 정도는 작업 중 재설정 가능 / 병렬 처리 설정은 Master Process에 의해 시행 / Oracle Database 11g Enterprise Edition에서만 사용 가능
   expdp HR/HR DIRECTORY=TEST_DIR DUMPFILE=SCHEMA_%U.DMP PARALLEL=2   
   SCHEMAS=HR,OE,SCOTT,SH JOB_NAME=SCHEMA_JOB

   SELECT OWNER_NAME,
       JOB_NAME,
       OPERATION,
       JOB_MODE,
       STATE
   FROM DBA_DATAPUMP_JOBS;
   
● Monitoring Job Status : Client 는 Logging Mode 또는 Interactive-command Mode 로 작업에 attach 할 수 있음 / Logging Mode 에서는 작업에 대한 실시간 세부 상태가 작업 실행 중에 자동으로 표시 / Interactive-command Mode 에서 작업 상태는 요청 시 표시 / Log File 은 작업 실행 중에 선택적으로 기록 / DBM_DATAPUMP_JOBS, USER_DATAPUMP_JOBS, DBA_DATAPUMP_SESSION 뷰 조회
 - Data Pump 작업은 작업 진행률을 나타내는 V$SESSION_LONGOPS 에 기록
 - 예상 전송 크기가 포함되며, 전송된 실제 데이터 양을 반영하도록 주기적으로 업데이트
 - COMPRESSION, ENCRYPTION, QUERY 및 REMAP_DATA 사용은 추정 값의 결정에 반영되지 않음

   SELECT sid, serial#, sofar, totalwork 
   FROM v$session_longops
   WHERE opname=‘JOB_NAME'
   AND sofar != totalwork ;

● Loading and Unloading of Data : 작업자 프로세스는 Meta Data와 Table Data를 Unload하고 Load합니다. Export의 경우 Transportable Tablespace를 사용하는 작업을 제외하고 모든 Meta Data와 Data가 병렬로 Unload 됩니다. Import의 경우 개체를 올바른 종속성 순서로 만들어야 함.
  
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
- Full 파라미터를 사용하며 데이터베이스 전체를 Export, DBA 권한이어야 가능, export_full_database 권한을 가지고 있어야 수행 가능
- DBA 권한을 가지고 있거나 Export_Full_Database 권한을 가지고 있어야 수행 가능.
2. Schemas -> 데이터베이스를 구성하는 레코드의 크기, 키의 정의, 레코드와 레코드의 관계, 검색 방법 등을 정의한 것
   export에서 owner 파라미터랑 같은 schemas parameter를 사용하여 특정 스키마 전체를 export 받음
- Schemas 파라미터를 사용하며 특정 스키마 전체를 Export
3. Table
- Tables 파라미터를 사용하여 여러 개의 테이블을 Export 받으려면 , 로 구분
4. Tablespace -> 테이블 및 인덱스를 저장해놓은 오라클의 논리적인 공간
- transport_tablespace 파라미터를 사용하면 테이블, 테이블 스페이스의 메타 데이터까지 export하여다른 서버로 테이블 스페이스 전체를 이동 시킬 때 아주 유용함 (단, 양쪽 OS, Block size, Characterset이 동일해야 함)
- Tablespace를 파라미터로 사용하며 해당 Tablespace에 속한 모든 Table을 받을 수 있다.


# Oracle Data Pump (+export/Import)
# Expdp(Export Data Pump) 실행 예제
Full Export
 expdp system/oracle dumpfile=full.dmp directory=dump full=y logfile=full.log job_name=fullexp
Metadata Export
 expdp system/oracle dumpfile=metadata.dmp directory=dump full=y content=metadata_only logfile=meta.log job_name=meta
Schemas Export
 expdp system/oracle dumpfile=test.dmp directory=dump schemas=TEST job_name=test logfile=test.log
Ex) Scott 계정의 emp, dept 테이블만 백업 받기
 expdp scott/tiger tables=emp,dept directory=datapump job_name=test1 dumpfile=emp_dept
Ex) Scott Schema 전부 백업 받기
 expdp scott/tiger schemas=scott directory=datapump dumpfile=scott.dmp
# Impdp 실행 예제
Full Import
 Impdp system/oracle dumpfile=full.dmp directory=dump full=y logfile=fullimp.log job_name=fullimp
Metadata Improt(Metadata만 export한 파일)
 Impdp system/oracle dumpfile=metadata.dmp directory=dump full=y sqlfile=metadata.sql logfile=metadata.log
Metadata Improt(Full export한 파일)
 Impdp system/oracle dumpfile=full.dmp directory=dump full=y content=metadata only sqlfile=metadata.sql logfile=metadata.log
Schemas Import
 impdp system/oracle dumpfile=full.dmp directory=dump schemas=TEST logfile=test.log job_name=test
 impdp system/oracle dumpfile=test.dmp directory=dump full=y logfile=test.log job_name=test
 
 # Impdp 실행 옵션
 데이터 적재에만 사용되는 옵션(TABLE_EXITS_ACTION) : export/import 기능에서 import의 ignore기능이 datapump에서는 TABLE_EXITS_ACTION 옵션이다.
 SKIP : 동일 이름의 테이블이 존재할 경우 해당 테이블에 대한 데이터 적재 작업 생략
 APPEND : 동일 이름의 테이블이 존재할 경우 데이터를 해당 테이블에 추가로 적재
 REPLACE : 동일 이름의 테이블이 존재할 경우 해당 테이블을 제거한 후 재생성하여 데이터 적재
 TRUNCATE : 동일 이름의 테이블이 존재할 경우 해당 테이블의 데이터를 DELETE한 후 데이터 적재 (★)
 
 # Attach 옵션
 ADD_FILE : 추출 파일(DUMPFILE) 추가
 CONTINUE_CLIENT : 데이터 펌프 작업에 대한 진행 로그 확인
 EXIT_CLIENT : 데이터 펌프의 클라이언트 관리 세션 종료
 KILL_JOB : 데이터 펌프 작업을 삭제(작업 재시작 불가능) (★)
 PARALLEL : 병렬 프로세싱 옵션 지정
 START_JOB : 데이터 펌프 작업 재시작 (★)
 STOP_JOB : 수행중인 데이터 펌프 작업 중단 (작업 재시작 가능)(★)
 
 # 작업 예상 시간 추출하기
 $ sqlplus /as sysdba
 SQL> select sid, serial#, sofar, totalwork
      from v$session_longops
      where opname='DATAPUMP2' <-Job_name을 대문자로 입력
      and sofar != totalwork;
     
 
 
 
 
