---
layout: post
title: "elasticsearch mapping 정리"
date: 2017-04-01 06:56:00 +0900
categories: elasticsearch
---

# elasticsearch mapping
elasticsearch 의 mapping 에 관한 내용을 정리하였다. 버전은.. 당시에 5.x 버전이 나오지 않을때라, 2.x버전 기준으로 1.x 버전과 비교하면서 작성하였다. elk 스택은 버전업이 너무 빠르다. 공식 문서보는게 가장 속편할듯 싶다.

## 목차
1. 개념
2. 매핑 생성 방법
3. 매핑 타입 종류
4. 매핑 파라미터들
5. Dynamic Mapping
5. References

## 개념
- 데이터베이스의 스키마와 같다고 할수 있음
- document 내의 필드의 데이터가 저장 및 색인되는 방법 정의
	- string 타입의 필드가 full text field로써 사용되는지
		- analyzer, tokenizer, tokenFilter 설정에 따라 다름
	- 숫자, 날짜, 지리적 정보의 타입으로써 사용되는지
	- 모든 필드([_all](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-all-field.html)) 의 인덱싱을 허용할 것인지
		- _all : 모든 필드의 텍스트를 모두 연결한 필드
	- 날짜 타입이면 날짜의 포맷은 어떻게 될 것인지
		- yyyy-mm-dd
	- 매핑 정보의 운영의 룰은 어떻게할 것인지
		- dynamic or strict
- elasticsearch 에서 type 내 도큐먼트 구분은 논리적이다. 모든 도큐먼트 타입은 같은 루씬 색인에 들어가지만 모두 분리되지 않는다. 타입은 순수하게 논리적인 개념이고, 수많은 레코드가 존재시 성능에 영향을 미친다. 모든 레코드가 동일한 색인 파일에 저장되기 때문에, 레코드의 읽기, 쓰기에 영향을 준다.
- 매핑을 세밀하게 조정하면
	- 디스크의 색인 크기가 줄어든다. (사용자 정의 필드의 기능을 비활성화)
	- 관심 있는 필드만 색인(성능 향상)
	- 빠른 검색, aggregation 등의 분석을 위해 데이터를 미리 색인한다.
	- 필드를 다중 토큰에서 분석할지, 필드를 하나의 토큰으로 분석할지 여부를 올바로 정의한다.

## 매핑 생성 방법
1. index 생성시

	```
PUT 인덱스명
{
	"mappings": {
        "타입명" : {
            "properties": {
                "필드명" : {
                    "type": "string",
                    "index": "not_analyzed"
                }
            }
        }
    },
    "settings": {
        "analysis": {
            ""
        },
        // shard, replica 지정
    }
}
```

2. `{index이름}/_mapping` api로 생성
	- 인덱스 존재시

		```
PUT <인덱스>/_mapping/<타입>
{
	"<타입명>" : {
		"<내장필드명>" : {
			… <필드 내용> …
		}
	}
}
		``` 

	- 매핑에 필드가 이미 존재하고, 필드값이 동일하면 : 매핑 변경하지 않는다.
	- 매핑에 필드가 이미 존재하고, 필드값이 다른 타입이면 : integer -> long 의 경우 필드 타입을 업그레이드할 것이지만, 타입이 호환되지 않으면 실패
	- 단, 이미 있는 필드를 제거할수는 없다.
		- Elasticsearch(및 Lucene)는 자신의 인덱스를 변경이 불가능한 세그먼트에 저장합니다 — 각각의 세그먼트는 "미니" 역 인덱스입니다. 이 세그먼트들은 적소에 있는 동안에는 절대 업데이트되지 않습니다. 도큐먼트를 업데이트하는 것은 실제로는 새로운 도큐먼트를 생성한 뒤 기존 도큐먼트는 삭제된 것으로 표시하는 작업입니다. 더 많은 도큐먼트를 추가(또는 기존 도큐먼트를 업데이트) 하게 되면 새 세그먼트가 생성됩니다. 백그라운드에서 실행되는 병합 프로세스에서 여러 개의 작은 세그먼트가 새로운 큰 세그먼트로 병합된 후 기존 세그먼트가 완전히 삭제됩니다.
		- [매핑 변경](https://www.elastic.co/kr/blog/changing-mapping-with-zero-downtime)

## 매핑 타입의 종류
### Meta-Fields
- 다큐먼트의 메타정보를 담는 필드

#### Identify meta-fields
- \_index : 다큐먼트가 속해있는 index
- \_type : 다큐먼트의 속해있는 index의 type
- \_id : 다큐먼트의 unique한 id
	- 색인시 사용자가 설정할수 있고, 없으면 elasticsearch 가 자동으로 할당
- \_uid : 색인 내의 유일한 식별자 \_type + \_id로 구성됨.

#### Document source meta-fields
- \_source : json 형태의 다큐먼트 바디
- \_size : \_source의 크기([mapper-size plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/2.4/mapper-size.html)에서 제공)

#### Indexing meta-fields
- \_all : 모든 필드의 텍스트를 모두 연결한 필드
	- default : true
	- 모든 필드의 텍스르를 잇기에 필드별로 의미부여를 다르게 해놓았더라도 Full Text Search 시 검색이 가능하다.
		- query string query 에서 필드명을 따로 지정하지 않을 경우 _all 필드로 검색(링크 : [_all 필드검색](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-all-field.html#querying-all-field))
	- CPU, 저장소를 많이 필요로 하다면 불필요시 false 로 변경하는 편이 좋음
- \_timestamp : 다큐먼트의 타임스탬프를 자동으로 색인
	- default : false
- \_ttl : 다큐먼트의 만료시간
	- default : false, default 시간을 설정할수 있음
- \_parent : 부모 다큐먼트 정의
- \_field_names : 다큐먼트가 가지고 있는 필드들의 이름

#### Routing meta-fields
- \_routing : 어느 샤드에 다큐먼트가 저장될지 제어
	- path : 라우팅에 사용될 필드
	- required : 라우팅 값 사용 여부 강제할 값

#### Fields or Properties
- type 내부의 필드명
	- elasticsearch 2.0~2.3 버전대에서 dots(.)이 field명에 포함될수 없도록 했는데, 2.4버전에서는 다시 dot을 허용함
		- [mapper.allow_dots_in_name=true 설정로 설정시 가능](https://www.elastic.co/guide/en/elasticsearch/reference/current/dots-in-names.html#_enabling_support_for_dots_in_field_names)


### 필드의 데이터타입
#### Core datatypes
- date : 날짜
- numeric
	- long : signed 64-bit integer 
	- integer : signed 32-bit integer
	- short : signed 16-bit integer
	- bype : signed 8-bit integer
	- double : double-precision 64-bit IEEE 754 floating point.
	- float : single-precision 64-bit IEEE 754 floating point.
- string : text values
	- 텍스트는 full text(analyzed) or keyword(not_analyzed) 으로 나뉠 것
	- full text
		- analyzer 속성을 설정하여 필요한 토큰들로 분해하여 색인하도록 설정
			- 하나의 [Tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-tokenizers.html)
				- 하나 이상의  [CharFilters](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-charfilters.html)
					- ex) html mark up -> strip out, & -> and 
			- 0개 이상의 [TokenFilters](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-tokenfilters.html)
		- analyzer를 설정하지 않으면 default로 [Standard Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-standard-analyzer.html)를 타게 됨.

			> 토큰이 어떻게 분석되는지 확인하려면 _analyze api 를 이용한다.
				```
				POST _analyze?analyzer=standard
{
    "한글은 어떻게 abc123 bcd 323"
}
```



#### Complex datatypes
- Array 타입

	```
	[ "a", "b", "c" ]
	```

- Object 타입

	```
	{
color: "red",
value: "#f00"
}
	```
- Nested 타입 : object로 이루어진 Array

	```
	"user" : [ 
{
  "first" : "John",
  "last" :  "Smith"
},
{
  "first" : "Alice",
  "last" :  "White"
}
]
	```
- elasticsearch에서 type을 명시해주지 않으면 default로 object 타입으로 저장됨
	- 위와 같은 구조의 데이터는 내부적으로 뎁스 없이 flat 하게 저장됨.

	```
	{
"user.first" : [ "alice", "john" ],
"user.last" :  [ "smith", "white" ]
}
	```
	- 그래서 관계가 엉키게 되는 문제가 발생하여 aggregation 에도 문제가 생김.
	- nested 타입으로 설정하게 되면,
		- queried with the [nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html).
		- analyzed with the [nested](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-nested-aggregation.html) and [reverse_nested aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-reverse-nested-aggregation.html).
		- sorted with [nested sorting](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-sort.html#nested-sorting).
		- retrieved and highlighted with [nested inner hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-inner-hits.html#nested-inner-hits).

#### Geo datatypes
- Geo-point datatype
	- geo_point for lat/lon points
- Geo-Shape datatype
	- geo_shape for complex shapes like polygons
- logstash [geoip](https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html) 이용할수 있을듯

#### Specialised datatypes
- IPv4 datatype
	- ip for IPv4 addresses
- Completion datatype
	- completion to provide auto-complete suggestions
- Token count datatype
	- token_count to count the number of tokens in a string
	- elasticsearch 2.x 버전부터 추가된 datatype
	- 토큰의 개수를 저장하는 데이터타입 같음.
- mapper-murmur3
	- murmur3 to compute hashes of values at index-time and store them in the index
	- [Mapper size plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/2.4/mapper-size.html) 이용 ; meta field 로 _size 가 추가되는데, 해당 다큐먼트가 인덱싱된 사이즈를 확인할 수 있다.
- Attachment datatype
	- See the mapper-attachments plugin which supports indexing attachments like Microsoft Office formats, Open Document formats, ePub, HTML, etc. into an attachment datatype.

## Mapping Parameters

### properties
- type / object / nested 등의 datatype 정보를 입력할수 있도록 하는 필드

### store
- 빠르게 검색할 수 있도록 분리한 색인 조각에 저장될 필드를 표시한다. 필드를 저장하면 디스크공간을 차지하지만, 스크립트, 집계에서 다큐먼트의 필드를 추출할 때 계산이 줄어든다.
- 기본적으로 \_source 필드가 저장되어있도록 되어있는데, 이를 false 로 했을때 
- 예를 들면 /index/type/1로 "서울특별시"라는 값을 가지는 title이라는 항목을 store="yes"로 속성을 지정후 색인 시키면 서울, 특별시라는 단어로 색인, 동시에 "서울특별시"도 저장된다. 검색시 저장된 값을 리턴받고 싶은경우 fields 파라미터에 지정하면 된다.
- [_source, store 용도](http://blog.naver.com/PostView.nhn?blogId=parkjy76&logNo=30172499283&parentCategoryNo=&categoryNo=68&viewDate=&isShowPopularPosts=false&from=postView)

### copy_to
- elasticsearch 2.x 에 추가됨
- \_all 필드를 customizing 해준다.
- 여러개 필드를 그룹핑할수 있게 함

### analyzer
- 텍스트 분석기

### search_analyzer
- 검색시 사용할 analyzer를 정의
- 정의하지 않으면 analyzer에 정의한 analyzer 혹은 standard analyzer 를 이용하게 된다.

### similarity
- elasticsearch 에서 사용할 scoring algorithm을 지정할 수 있다.
- 이는 analyzed된 string datatype에서 유용하다.
- default : TF/IDF / BM25

### inclue\_in\_all
- \_all 필드를 색인할지의 여부를 결정(default : true)

### index
- 해당 필드를 색인할지를 결정
- analyzed or not_analyzed or no

### index\_options
- elasticsearch 2.x 에 추가됨
- docs
	- Only the doc number is indexed. Can answer the question Does this term exist in this field?
- freqs
	- Doc number and term frequencies are indexed. Term frequencies are used to score repeated terms higher than single terms.
- positions
	- Doc number, term frequencies, and term positions (or order) are indexed. Positions can be used for proximity or phrase queries.
- offsets
	- Doc number, term frequencies, positions, and start and end character offsets (which map the term back to the original string) are indexed. Offsets are used by the postings highlighter.
	- stirng datatype 은 offsets 가 default 옵션이라, 쿼리 옵션에 highlight를 주면 해당 offset에 효과를 줄수 있음

### null_value
- 필드를 못찾을 경우에 쓰일 default 값을 정의
- 기본적으로 필드에 값이 없을 경우엔 인덱싱/검색이 되지 않으나, default 값을 정의하면 그 값은 인덱싱/검색이 가능하게 된다.

### fields
- elasticsearch 2.x 에 추가됨
- 하나의 필드에 여러가지 인덱싱 방법을 둘수있음.
	- string datatype 하나의 필드에 analyzed, not_analyzed의 indexing 속성을 두어 aggregation 할때 두개를 적절하게 번갈아가면서 사용가능

### boost
- 필드의 스코어를 인덱싱 시간에 boosting할수 있게됨.(default : 1.0)
- 그러나 인덱싱 시간에 boosting하는 것은 bad idea라고 함
	- 다큐먼트를 재색인하지 않으면 부스팅값을 바꿀수 없음
	- 쿼리에서 필드별 boosting 을 할수 있음

### coerce
- elasticsearch 2.x 에 추가됨
- 값을 필드에 맞게 바꿔준다.
	- "1" -> 1
	- datatype이 integer 인 경우 소숫점은 탈락
	- lon/lat 지리적 정보 -> -180:180 / -90:90으로 제한

### dynamic
- 새로운 타입, 필드를 PUT 하면, 필드의 값에 맞는 데이터타입의 default값으로 반영됨
- 타입레벨에서 설정할수도 있고, 필드레벨에서도 설정할 수 있다. 
- true : default, 새로운 필드를 감지하여 매핑에 추가한다.
- false : 새로운 필드를 ignore 한다, 새로운 필드는 mapping api를 통해 명시적으로 등록해야한다.
- strict : 새로운 필드를 감지하면, exception을 뱉는다.

### enabled
- elasticsearch 2.x 에 추가됨
- elasticsearch 는 기본적으로 모든 필드를 인덱싱하는데, 때로는 인덱싱은 하지 않고 저장만 하는 필드를 원할수 있다.
- enable로 세팅에 의해 이를 결정할 수 있다.
- 색인 되지 않고, 필드에 대한 정의(integer, object, nested 타입인지) 하지 않아도 된다.

### doc_values
- on-disk data structure / query-time data structure
	- on-disk data structure 
		- built at document index time, which makes this data access pattern possible.
	- query-time data structure
		- built on demand the first time that a field is used for aggregations, sorting, or is accessed in a script. 
- Doc values are the on-disk data structure, built at document index time, which makes this data access pattern possible.
- analyzed string field 제외하고, 나머지는 다 가능하다.
- disk에 저장하는 방식이므로, heap 의 사용량이 줄어든다.
- false 로 하면
	- sort, aggregation되지 않음
	- 스크립트 로 값을 access하지 못함
	- 디스크 공간이 절약됨

### fielddata
- elasticsearch 2.x 에 추가됨
- doc_values 가 analyzed string field를 지원하지 못하는데, fielddata는 query-time data structure 로서 이를 지원한다.
- 처음 aggregations, sorting, 스크립트를 통한 데이터 액세스를 하게 되면 heap 에 데이터를 캐싱하게 된다.
- fielddata.format , fielddata.loading , fielddata.filter 의 설정필드들이 있다.
- doc_values와는 반대로, heap 사용량이 커진다.

### format
- date datatype 을 위한 포맷을 정의할 수 있다.

### ignore_above
- string datatype 에서 글자수를 제한해서 색인할 수 있도록 하는 기능이다.
- not_analyzed인 string 필드에 ignore_above 를 설정하게 되면 해당 길이가 넘어간 term 은 색인되지 않는다.

### position\_increment\_gap
- analyzed 된 string datatype 은 term 의 position 을 다루게 되어있는데, 이 gap을 mapping설정을 통해 줄수도 있다.
- default : 100

## Dynamic Mapping
### ```_default_``` Mapping
- index 생성시 하위 타입 및 필드들에 default 값들을 지정할 수 있다.
- dynamic_templates
	- 필드들의 속성들을 템플릿화 할수 있다.
- [링크](http://opens.kr/90)를 통해 간단한 예제를 확인할 수 있다.

### Override default template
- _template api 를 통해 mapping 자체를 탬플릿화 할 수있다.



## References
- [시계열 데이터 스토어로서의 Elasticsearch](https://www.elastic.co/kr/blog/elasticsearch-as-a-time-series-data-store)
- [Field Data: The Most Common Cause of Elasticsearch Cluster Instability at Scale](https://qbox.io/blog/field-data-elasticsearch-cluster-instability)
- [Support in the Wild: My Biggest Elasticsearch Problem at Scale](https://www.elastic.co/blog/support-in-the-wild-my-biggest-elasticsearch-problem-at-scale)