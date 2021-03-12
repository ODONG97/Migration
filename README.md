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
