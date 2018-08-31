---
layout: post
title: "일래스틱서치 고급 기능의 개념과 활용 7장 정리"
date: 2018-06-12 13:30:00 +0900
categories: elasticsearch
---

# 7장 로우레벨 인덱스 제어
## 아파치 루씬 스코어링 변경
앞서 2장에서 다루었던 BM25, TF-IDF 외에 사용할 수 있는 스코어링 모델에 대해 설명하고 있다. elasticsearch 에서 참고할만한 문서는 [링크](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html)를 참고

### 사용가능한 유사도 모델
- TF-IDF : elasticsearch 5.0 이전 버전은 default, 5.0 이후 버전에서 사용하려면 유사도 모델을 "classic"으로 선언
- DFS(divergence from randomness)
- DFI(divergence from independence)
- IB(information-based)
- LM 디리클레(LM Dirichlet)
- LM 옐리네크 머서(LM Jelinek Mercer)

### 필드마다 유사도 설정
```
{
	"mapping": {
		"post": {
			"properties": {
				"contents": {
					"type": "text",
					"store": "no",
					"index": "analyzed",
					"similarity": "classic" // TF-IDF 적용
				}
			}
		}
	}
}
```

### 유사도 모델 설정

```
{
	"setting": {
		"index": {
			"similiarity": {
				"mastering_similarity": { // 설정에 랭킹모델을 설정할 수 있음
					"type": "classic",
					"discount_overlaps": false
				}
			}
		}
	},
	"mapping": {
		"post": {
			"properties": {
				"contents": {
					"type": "text",
					"store": "no",
					"index": "analyzed",
					"similarity": "mastering_similarity"
				}
			}
		}
	}
}
```
- TF-IDF
	- discount_overslaps : 스코어 계산시에 0으로 설정된 위치 증가를 포함한 토큰은 고려하지 않고 있음. 0 스코어를 고려하고 싶다면 이 값을 false로 설정
- BM25
	- k1 : 포화-비선형 단어 빈도 정규화
	- b : 도큐먼트 길이가 단어빈도에 미치는 영향
	- discount_overlaps
- 이외에 DFR, IB, LM디리클레,LM 옐리네크 머서도 설정할 수 있는 값들이 있음.

##저장소 모듈
저장소 모듈은 I/O 하위 시스템과 루씬사이의 추상화이다. 루씬이 저장장치를 이요해 수행하는 모든 작업은 저장소 모듈을 사용하여 수행함. 저장소 모듈에 대한 elasticsearch 문서는 [링크](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html)를 참고
elasticsearch의 대부분 저장소 타입은 적절한 [루씬 디렉토리](http://lucene.apache.org/core/7_3_1/core/org/apache/lucene/store/Directory.html)에 매핑됨

### 저장소 타입
- 설정방법
	- elasticsearch.yml 파일에 추가
		- index.store.type: niofs
	- 인덱스 생성시 인덱스의 저장소 타입 설정
	
		```
		{
			"setting": {
				"index.store.type": "niofs"
				}
			}
		}
		```
		
- simplefs
	- java.io.RandomAccessFile 클래스를 사용해 구현된 Directory 클래스 구현을 사용
	- 루씬의 [SimpleFSDirectory](http://lucene.apache.org/core/7_3_1/core/org/apache/lucene/store/SimpleFSDirectory.html)와 매핑
	- 간단한 어플리케이션에서는 simplefs 만으로 충분.
	- 주요 병목 : 다중 쓰레드가 접근할때 성능이 떨어짐.
- niofs
	- java.nio.FileChannel 클래스를 기반으로 하는 Directory 클래스 구현을 사용
	- 루씬의 [NIOFSDirectory](http://lucene.apache.org/core/7_3_1/core/org/apache/lucene/store/NIOFSDirectory.html)와 매핑
	- 성능 저하 없이 여러 쓰레드가 동일한 파일에 동시에 접근할 수 있도록 함
- mmapfs
	- MMapDirectory 구현을 사용
	- read -> mmap 시스템 호출
	- write -> RandomAccessFile 사용
	- 매핑된 파일의 크기가 동일한 프로세스에서 사용 가능한 가상 메모리 주소 공간의 일부를 사용
	- 잠금 기능이 없으므로 다중 쓰레드가 접근할 때 확장이 가능
	- OS의 인덱스 파일을 읽기 위해 mmap 사용시 이미 캐싱된 것처럼 보임. 이때문에 루씬 인덱스에서 파일을 읽을 때 OS 캐시에 읽을 필요가 없어서 접근이 더 빠름. 
	- 기본적으로 루씬과 elasticsearch가 IO 캐시에 직접 접근할 수 있어서 인덱스 파일에 빠르게 접근할 수 있음
	- 64비트 환경에서 가장 잘 동작하고, 32비트 환경은 인덱스가 충분히 작고, 가상 주소 공간이 충분하다고 확신할 때만 사용
- fs
	- 기본 파일 시스템 구현. elasticsearch는 운영체제 환경에 따라 최적의 구현을 선택
	- 윈도우 32비트 -> simplefs, 윈도우 외 -> niofs
	- 64비트 -> mmapfs

## NRT, 플러시, 리프래시, 트랜잭션 로그
### 인덱스 변경과 변경사항 커밋
```
curl -XPOST localhost:9200/test/test/1 -d '{"title":"test"}' ; 
curl -XGET localhost:9200/test/test/_search
```
- 커밋 과정
	- 인덱스가 생성중일때 새로운 도큐먼트는 세그먼트로 저장
	- 도큐먼트 저장과 동시에 병행적으로 실행되는 쿼리는 새로 만든 세그먼트를 검색할 때 사용되는 세그먼트 집합에 추가해야함
	- 루씬은 인덱스의 세그먼트를 나열하는 segments_N 파일을 연속해서 생성
- refresh
	- 루씬은 인덱스에 접근하기 위해 Search를 사용하는데 이 클래스가 리프레쉬되어야함.
	- 새로 작성한 세그먼트를 보기 위해서는 Search 객체를 새로 연다
	- 어플리케이션은 이 리프레쉬 작업이 몇초에 한번 수행될지 정할 수 있음.

### 기본 리프레시 시간 변경
```
PUT /test/_settings
{
	"index": {
		"refresh_interval": 5m
	}
}
```

### 트랜잭션 로그
- 루씬은 인덱스의 일관성을 보장할 수 있음
- 그러나 데이터를 인덱스에 쓰는 동안 에러가 발생할 때 데이터 손실이 발생하지 않도록 보장할 수는 없음
- 빈번한 커밋은 성능 면에서 비용이 많이 발생(커밋 한번당 새로운 세그먼트가 생성)
- 트랜잭션 로그에 커밋되지 않은 트랜잭션들이 저장되고 후속 변경사항에 대한 새로운 로그를 만듬
- flush : 트랜잭션 로그 정보가 저장소와 동기화되고 트랜잭션 로그가 지워짐
- 트랜잭션 로그 설정
	- index.translog.sync_interval(5s) : 트랜잭션 로그가 디스크에 얼마나 자주 동기화되는지 제어
	- index.translog.durability
		- 트랜잭션 로그를 fsync, 커밋 수행 방법 설정
		- request (default)
			- 요청이 있을 때마다 fsync, 커밋작업 수행
			- 하드웨어 에러 발생시 승인된 모든 저장은 디스크에 커밋
		- async
			- 요청이 있을 때마다 fsync, 커밋작업은 sync_interval 간격마다 백그라운드로 수행
			- 하드웨어 장애 발생시 마지막 자동 커밋 이후 승인된 모든 저장작업은 삭제됨
	- index.translog.flush\_threshold\_size: 트랜잭션 로그의 최대 크기
	- 이 값들은 `PUT /{인덱스명}/_settings `를 통해 동적으로 제어 가능
- 손상된 트랜잭션 로그의 처리
	- 트랜잭션 로그 파일의 checksum 이 불일치하는 현상이 감지되면 elasticsearch 는 해당 샤드를 장애로 간주함.
	- 데이터를 복구할 수 있는 사본이 없다면 현재 트랜잭션 로그를 잃어버리는 비용을 지불하고 샤드에 포함된 데이터를 복구할 수 있음
		- elasticsearch-translog 스크립트를 통해 진행 가능
		- elasticsearch 가 실행중일때 해당 스크립트를 실행하면 안됨. 해당 스크립트 실행시 트랜잭션 로그에 포함된 도큐먼트가 영구적으로 손실될 것
		`/{elasticsearch 설치 경로}/bin/elasticsearch-translog truncate -d {손상된 인덱스 경로}/0/translog/`

### NRT(near-realtime search) GET
- 실시간으로 제공
- 커밋되지 않은 버전을 포함해 이전 버전의 도큐먼트로 돌아갈 수 있는 기능 제공
- 도큐먼트의 최신버전이 트랜잭션 로그에 존재한다면 그 버전의 도큐먼트가 리턴됨.

### 세그먼트 병합의 제어
- 세그먼트 병합
	- 루씬 세그먼트는 한번만 저장되고 여러번 읽히는 구조인데, 세그먼트 파일 중 하나에 포함된 삭제 도큐먼트를 제외하고 데이터 구조가 변경될 수 있다. 
	- 일정시간이 지나 특정조건 충족시 일부 세그먼트를 더 큰 세그먼트로 복사하고, 원래의 세그먼트는 디스크에서 삭제됨
- 세그먼트가 성능에 미치는 영향
	- 세그먼트는 많아지면 많아질수록 검색 속도가 느려지고 루씬이 메모리를 더 많이 필요로 함
	- 세그먼트 자체는 불가능하기 때문에 세그먼트에서 정보를 삭제할 수 없음
	- 세그먼트 병합시 삭제된 도큐먼트는 기록되지 않고 제거되기 때문에 병합 완료시 전체 인덱스의 크기가 줄어듬

### 세그먼트 병합 정책 변경
- elasticsearch 2.x 버전까지는 병합조절에 대한 throttle 값이 있었는데 제거됨

### 계층 계층병합 정책 설정
- 계층 병합정책은 기본값, elasticsearch 5.x에서 사용하는 유일한 병합정책
- 계층별로 허용되는 최대 세그먼트 개수를 고려하여 거의 비슷한 크기의 세그먼트를 병합함
- 계층 병합 정책 옵션
	- index.merge.policy.max\_merge\_at\_once : 저장되는 동안 동시에 병합될 최대 세그먼트 개수
	- index.merge.policy.segments\_per\_tier : 계층당 허용되는 세그먼트 개수
	- 이외 계층 병합 정책 옵션들은 책을 참고

### 스케쥴링 병합
- elasticsearch 는 세그먼트 병합을 [ConcurrentMergeScheduler](https://lucene.apache.org/core/7_3_1/core/org/apache/lucene/index/ConcurrentMergeScheduler.html)를 사용함
- 병합 허용 최대 쓰레드 개수는 조절 가능

### 강제 병합
- 세그먼트를 병합하여 세그먼트 개수를 줄일 수 있음
- 전체 세그먼트 개수를 1로 한다면 검색속도는 매우 빠를 것
- 병합시 병합 완료되기 전까지 새로운 요청은 차단되고, 병합시간은 오래걸릴 수 있음

## elasticsearch 캐시
### node query cache
- 쿼리 결과를 캐싱함
- 노드마다 하나의 쿼리 캐시가 있고, 해당 노드에 존재하는 모든 샤드에 의해 공유됨
- LRU 캐시 정책 사용

### shard request cache
- 각 샤드별 hits.total, aggregation, suggestions 결과를 캐싱함
- 코디네이터 노드에 검색 요청 실행시 모든 데이터노드에 요청을 전달하여 코디네이터로 검색결과를 리턴하면 코디네이터노드는 샤드레벨 결과를 집계함(aggregation 도 수행)

### field data cache
- 데이터 기반으로 동작하는 오퍼레이션을 포함하는 쿼리 수행에 필드데이터 캐시가 이용됨
- aggregation, script, sort 사용시 elasticsearch는 해당 캐시를 사용

### 서킷 브레이커 사용
쿼리는 elasticsearch의 자원에 많은 부담을 줄수 있기 때문에 특정 기능에서 너무 많은 메모리를 사용하지 못하게 하도록 설정하는 기능. 메모리 사용의 특정 임계값 도달시 쿼리 실행을 거부함

- 부모 서킷 브레이커
	- indices.breaker.total.limit(기본 70%)
- 필드 데이터 서킷 브레이커
	- 요청에 대한 예상 메모리 사용량이 설정값보다 높은 경우 요청 실행 방지
	- indices.breaker.fielddata.limit(기본 60%)
- 요청 서킷 브레이커
	- indices.breaker.request.limit(기본 60%)
	- indices.breaker.request.overhead(기본 1)
- 인플라이트(in-flight) 요청 서킷 브레이커 요청
	- elasticsearch 가 현재 전송 중인 모든 수신 요청의 메모리샤용량, 특정 노드의 특정 메모리 양을 초과하지 않도록 HTTP레벨을 제한할 수 있음
	- 메모리 사용량은 요청 자체의 콘텐츠 길이를 기반으로함
	- network.breaker.inflight_requests.limit(기본 100%)
	- network.breaker.inflight_requests.overhead(기본값 1)
- 스크립트 압축 서킷 브레이커
	- 새로운 스크립트를 만나면 해당 스크립트를 컴파일하여 캐싱함. 컴파일작업이 무거울 수 있는데 일정 시간 내에 인라인 스크립트 컴파일을 제한함
	- 1분 내에 컴파일할 수 있는 동적 스크립트 개수를 제한할 수 있음
	- script.max\_compilations\_per\_minute(기본 15)