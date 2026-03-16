---
layout: post
title: "MetaData Lock"
date: 2025-01-19
description: "MySQL의 Metadata Lock이 무엇인지, 어떻게 발생하고 해결하는지를 실제 예제와 함께 정리한 글입니다."
tags: [Mysql]
tistory_id: 52
---

# 개요

![Metadata Lock Overview](https://blog.kakaocdn.net/dna/d74Kif/btsLTvlvKcx/AAAAAAAAAAAAAAAAAAAAAByPcrnvCebVDI4Dbcf7mtTL4E9VRrIF1AxeSnht2KT3/img.png)

Metadata Lock이 무엇인지 설명하기 전에 해당 락이 발생하면 어떻게 되는지 보여주는 예시입니다.

해당 Table을 Select할 때 Lock이 있기 떄문에 조회가 안되는 현상을 보고 있습니다.

하지만 Lock이 안 걸린 다른 Table을 조회할려할 땐 됩니다.

![Lock Example](https://blog.kakaocdn.net/dna/tjhcM/btsLTOLTUhu/AAAAAAAAAAAAAAAAAAAAAIevwjsbn2Fv2D4ecA3K2kc0o14NYBPO66HiAbdRqirU/img.png)

이 MetaData Lock이 뭔지 알아가봅시다.

# Metadata Lock

## Metadata Lock?

테이블, 스키마, 저장된 프로그램(프로시저, 함수, 트리거, 예약된 이벤트 등), 테이블 스페이스, 함수로 획득한 사용자 잠금이다. 여러 세션에서 해당 락을 가질려면 우선 순위에 따라 락을 획득할 수 있다.

참고로 해당 예제는 테이블 변경으로 인한 예제이다.

## Metadata Lock 획득

- **Read Lock (record 단)**
  - DML 문을 실행할 때 획득하는 락, 배타적이지 않아 여러 세션에서 Read Lock을 획득할 수 있다.

- **Write Lock (Table 단)**
  - DDL 문을 실행할 때 획득하는 락, 배타적이어서 하나의 세션에서만 Lock을 획득할 수 있다.
  - 해당 락 이후 들어오는 세션들을 Blocking 한다.

## Metata Lock 해제

Lock을 가진 Transaction이 Commit이나 Rollback 행위를 하면 Lock을 반납한다.

## Metadata Lock Issue 구현

![Issue Step 1](https://blog.kakaocdn.net/dna/bYNwEq/btsLSTmWf79/AAAAAAAAAAAAAAAAAAAAAK2x4q1Qu2TXPNV7ypkJP8-qtRGRH5WcrZ6dZx-ltKZS/img.png)

![Issue Step 2](https://blog.kakaocdn.net/dna/c74gGU/btsLSfxypl4/AAAAAAAAAAAAAAAAAAAAANdoQQBXau-HLI0zf05CEeluV2rgi1hBA7XafhZBWf2e/img.png)

![Issue Step 3](https://blog.kakaocdn.net/dna/k1Ohu/btsLSAaaPmi/AAAAAAAAAAAAAAAAAAAAAK94ag7tFS4vsRw7Y4E_bHriArSDFDFowR1Xxtc55T2T/img.png)

Transaction을 열고 DML문을 사용하여 Read Lock을 획득했지만 Commit or Rollback 안하고 대기 중임.

![Issue Step 4](https://blog.kakaocdn.net/dna/wcMfe/btsLSqLZ6By/AAAAAAAAAAAAAAAAAAAAADl3MtNL-TA3Z45XyvYJQFCGEu1YhWhLre423HXuqcJF/img.png)

다른 세션에서 DDL 문을 사용하여 Write Lock을 획득하고 실행함.

![Issue Step 5](https://blog.kakaocdn.net/dna/zQkNO/btsLSkL7GR7/AAAAAAAAAAAAAAAAAAAAAI713IVvN9GN2ztUWZsBXF8h8JybmkbhoI7mq-2iuR4s/img.png)

Show processlist 를 하여서 Waiting for table metadata lock 확인할 수 있음.

![Issue Step 6](https://blog.kakaocdn.net/dna/bb2Vkn/btsLS4BFqbv/AAAAAAAAAAAAAAAAAAAAAFBxPS_OULdYZEXS-_FmwLDbpywYZPT-O8vbPC3A2uAN/img.png)

![Issue Step 7](https://blog.kakaocdn.net/dna/rUngB/btsLSsbXLDn/AAAAAAAAAAAAAAAAAAAAAHXj14LovBgJzOGP8OT8m6_qD9as-aH8X9DVx4aSEl1Q/img.png)

Write Lock 이후 DML 을 실행했지만 Waiting for table metadata lock 을 걸린 모습을 확인

## Metadata Lock Issue 해결

실행 중인 Transaction을 Commit or Rollback을 하여 종료시키거나 Kill을 하여 종료시켜 주면 된다.

### Transaction을 알 때

![Solution Step 1](https://blog.kakaocdn.net/dna/0gcpD/btsLRj1nMRv/AAAAAAAAAAAAAAAAAAAAAEDJgfpxmo2aIrVixa9VKaDSwTDCWPys1jMTw11YIwdU/img.png)

해당 Transaction을 Commit 행위를 하여 종료시킴

![Solution Step 2](https://blog.kakaocdn.net/dna/7sI7Z/btsLS5UT3pb/AAAAAAAAAAAAAAAAAAAAAH0ylzDtyLGB6kp78-gPMuSnF3M4nnpevpQDQ4pVSTJJ/img.png)

Wait 된 다른 세션들이 제대로 종료된 걸 확인할 수 있다.

### Transaction을 모를 때

![Solution Step 3](https://blog.kakaocdn.net/dna/DXgUF/btsLSo12Ze5/AAAAAAAAAAAAAAAAAAAAAJkL6IufO-gYJS8CmLVtJaNGVOY7CyoqPszQe378CpL3/img.png)

`select * from information_schema.Innodb_trx;` 실행하여 현재 실행 중인 트랜잭션을 확인할 수 있다.

실행 중인 트랜잭션을 죽인다. `kill {trx_mysql_thread_id}`

![Solution Step 4](https://blog.kakaocdn.net/dna/bBxzzW/btsLSzPPQBK/AAAAAAAAAAAAAAAAAAAAAJp9v3OpoysyovZ_hIqdy1G99xwpvmCIiMPDc2svVp21/img.png)

병목이 해소된 걸 확인할 수 있다.

# 정리

Metadata lock은 주로 DDL을 변경할 떄 안전하게 변경하기 위한 Lock이다. 해당 락의 범위로는 Table 락 or Recode 락으로 존재한다. 만약 해당 Metadata Lock이 발생한 이후 병목이 해소가 안되면 실행 중인 Transaction을 찾아서 삭제하자!

제일 좋은 방법은 놀고 있는 Transaction을 발생시키면 안되지만 해당 Transaction을 만드는 주체는 어플리케이션 단에서 발생시키고 있기 때문에 찾아내기가 힘들다.
