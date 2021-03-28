# MapReduce: Simplified Data Processing in Large Clusters

## Abstract
MapReduce는 큰 데이터를 처리하고 생성하기 위한 프로그래밍 모델 및 이에 연관된 구현을 말한다. MapReduce 모델에서 map 함수는 하나의 key/value pair를 data를 매개(intermediate) key/value pair로 가공하는 역할을, reduce 함수는 매개 key와 관련된 모든 값들을 하나의 값으로 합치는 역할을 한다.

MapReduce 방식으로  작성된 프로그램은 자동으로 병렬화되어 큰 규모의 클러스터에서 실행이 된다. 프로그램이 돌아가는 런타임 환경에서 input data 파티셔닝, 프로그램 실행 스케쥴링, 장애 처리, 기계 간 통신 등을 처리해주기 때문에 병렬 및 분산 처리 시스템에 경험이 없는 개발자도 분산 처리 시스템을 활용할 수 있다.

## 1 Introduction
지난 몇 년 간 구글은 많은 양의 데이터를 처리하기 위해 수많은 계산법들을 구현했다. (크롤링된 문서, request 로그 등을 호스트 별로 크롤링된 페이지 개수, 특정 기간 동안 가장 자주 발생하는 쿼리 집합 등으로 변환)  데이터 처리 방법은 개념적으로 단순했지만, 데이터의 규모가 크기 때문에 합리적인 시간 안에 작업을 마치기 위해선 분산 작업이 필요했다. 이는 연산 병렬화, 데이터 분산, 장애 처리에 대한 이슈가 생겨, 많은 양의 코드를 요하기 때문에 간단한 연산을 어렵게 만들었다.

이런 복잡함에 대응하기 위해  실행하고자 하는 간단한 계산은 잘 표현하고, 병렬화, 데이터 분산, 장애 처리, 로드 밸런싱으로 인해 야기되는 복잡함은 숨겨주는 새로운 개념을 설계했다. 우리는 대부분의 연산이 record를 매개 키/값 쌍으로 바꾸어 주는 *map* 연산, 산출된 데이터를 합치기 위한 *reduce* 연산이 포함된다는 것을 알게 되어, 사용자가 map/reduce 함수를 정의하는 모델을 만들었다. 이 모델을 사용함으로써 우리는 연산을 쉽게 병렬화할 수 있었고, 본래의 매커니즘이 fault-tolerant하게 되어(?) 재실행(re-execution)이 가능해졌다.

이 작업은 모델을 구현함으로써 자동 병렬화와 데이터 분산이 결합을 가능하게 하는, 대규모 클러스터에서도 높은 성능을 나타내는 단순하고도 강력한 인터페이스를 만들었다는 점에서 의의가 있다.

## 2 Programming Model

MapReduce 모델은 키/값 쌍을 입력 받아 키/값 쌍 집합 형태의 Output을 생산한다. MapReduce 모델 사용자는 Map, Reduce 두 함수로 연산 작업을 표현한다.

Map은 입력 쌍을 받아 매개 키/값 쌍의 집합을 만든다. MapReduce 라이브러리는 같은 Key 값을 갖는 쌍을 그룹 지어 Reduce 함수로 보낸다.

Reduce 함수는 Key 값 하나와 그에 따른 Value의 집합을 입력값으로 받아, value를 더 작은 집합으로 만들기 위해 통합하는 작업을 한다. 대개는 각 reduce 작업마다 0 또는 하나의 값이 생산된다. 중간중간 들어오는 값은 iterator를 통해 제공된다. 이러면 메모리 대비 너무 큰 사이즈의 value들도 처리를  할 수 있게 된다.

### 2.1 Example

너무 유명한 word count 예제.
map 함수에서 문서를 단어 / 단어의 발생횟수로 쪼개어 반환하고, reduce 함수에서 특정 단어에 대한 횟수를 모두 더한다.

추가적으로, 사용자는 Mapreduce Specification Object를 생성하기 위해 input, output 파일 이름, 매개 변수 값 설정 등을 작성한다. 그 다음 사용자는 객체에 함수를 넘겨줌으로써 MapReduce 함수를 실행한다. 

### 2.2 Types
**map**		(k1, v1)		→ 	list(k2, v2)
**reduce**	(k2, list(v2)) 	→	list(v2)

### 2.3 More Examples
PASS!

## 3 Implementation

### 3.1 Execution Overview
![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/68cc8f06-db18-4b36-af9e-25f5cf36fc22/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/68cc8f06-db18-4b36-af9e-25f5cf36fc22/Untitled.png) 

### 3.2 Master Data Structures
Master는 각각의 Map Task와 Reduce Task에 대해 그 Task의 상태(idle, in-progress, completed)와, (할당이 완료된 Task에 한하여)  Worker Machine의 ID 정보를 가진다.
Master는 Map Task → Reduce Task로 중간 파일이 전파되는 통로이기 때문에, Map Task로 부터 만들어진 R개의 intermediate 파일 위치와 크기를 수집한다. 이 정보는 map task가 완료될 때마다 업데이트 되며, in-progress 상태의 reduce worker들에게 증분적으로 전송된다.

### 3.3 Fault Tolerance
**Worker Failure**

Master는 주기적으로 ping을 날린다. 만약 일정 시간 동안 worker로 부터 응답이 없을 경우, master는 해당 worker를 failed로 표시한다. Worker에 의해 완료된 Map Task는 다시 스케쥴링 될 수 있도록 idle한 상태로 리셋된다. (이게 Task를 Data만 달리하여 재탕한다는 말....인지?) 비슷하게, 고장난 worker에서 실행 중이던 Map 혹은 Reduce Task도 다시 스케쥴링 될 수 있도록 idle 상태로 리셋된다.

어떤 worker에 장애가 났을 때, 그 worker에서 완료되었던 map task들은 모두 다시 실행된다. Map Task의 Output file은 worker의 local disk에 저장되는데, worker에서 장애가 나버리면 파일에 접근할 수 없기 때문이다. 하지만 Reduce Task는 (worker가 고장나더라도) 다시 실행될 필요가 없다.

Map Task가 worker A에서 실행되었다가 A의 장애로 인해 worker B에서 재실행된 경우, Reduce Task를 실행하는 모든 Worker는 이런 상황을 통보받는다. 따라서 Map Task의 Output file을 Worker A로 부터 아직 못 읽어들인 Reduce Task가 있다면, 그 Task는 worker B에서 data를 읽어올 것이다.

MapReduce는 대규모의 worker 장애에 대해서 회복 탄력성이 있다. 만약 네트워크 장애로 인해 한번에 80대의 machine에 접근하지 못하게 될 때에도, MapReduce의 Master에서 장애가 발생한 worker들이 완료했던 Map Task들을 단순히 다시 실행시켜주기 때문에 연산을 완료할 수 있다.

**Master Failure**

Master가 관리하고 있던 Data를 주기적으로 체크포인트를 작성하는 것은 어려운 일이 아니다. 만약 Master Task가 죽으면, 가장 최근의 체크포인트 지점으로 Master Task의 복사본을 만들 수 있다.

하지만, Master가 하나인 것을 고려하면, 이런 장애는 쉽게 일어나지 않는다. (백 대 중에 한 대 고장나는 것보다야 한 대 중에 하나 고장날 확률이 적다는 뜻인듯...) 따라서 Master에서 장애가 발생하면 그냥 MapReduce 작업을 중단하도록 설계하였다.

**Semantics in the Presence of Failures**

사용자가 정의한 map과 reduce 함수가 입력 값에 대해 *deterministic function일 때, 분산 처리 구현은  전체 프로그램이 실패 없이 sequential하게 동작할 때와 동일한 결과를 생성한다.

이러한 속성을 달성하기 위해  Map과 Reduce의 *원자적 커밋(atomic commit)에 의존한다. 각 In-progress 상태의 Task는 개별 임시 파일에 자신의 결과를 작성한다.  Reduce  task는 하나의 임시파일을 생성하고 Map Task는 Reduce task 당 하나씩, 총 R개의 임시파일을 생성한다. Map Task가 완료되면, worker는 R개의 임시파일 정보가 담긴 메시지를 master에게 보낸다. Master가 이미 완료가 된 worker의 정보를 받으면 그 메시지를 무시하고, 아니면 R개의 파일 정보를 master의 data structure에 저장한다. Reduce Task가 완료되면, worker는 원자적으로 임시 파일의 이름을 최종 출력 파일로 바꾼다. 같은 Reduce Task가 여러 대의 machine에서 수행될 수도 있는데, 이 경우 이름 변경 호출도 여러 차례 수행이 된다. 이 때 파일 시스템의 원자적인 이름 변경 호출에 의존하여 오직 한번의 reduce task에서 실행된 결과만 유지하는 것을 보장한다.

보통은 map, reduce 연산은 deterministic하다. 하지만 non-deterministic한 경우에도 약하지만 꽤 타당한 의미를 제공하는데, ........

When the *map* and/or *reduce* operators are non-deterministic, we provide weaker but still reasonable semantics. In the presence of non-deterministic operators, the output of a particular reduce task R1 is equivalent to the output for R1 produced by a sequential execution of the non-deterministic program. However, the output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program.

Consider map task M and reduce tasks R1 and R2. Let e(Ri) be the execution of Ri that committed (there is exactly one such execution). The weaker semantics arise because e(R1) may have read the output produced by one execution of M and e(R2) may have read the output produced by a different execution of M.

엄청 중요하다기보다 문단이 잘 이해가 안 가서 원문 +  제 생각을 표현한 그림(...) 으로 대체합니당..

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7be65b66-3145-44ac-86ae-cdc9e11ed492/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7be65b66-3145-44ac-86ae-cdc9e11ed492/Untitled.png)

※ deterministic function: 특정 입력 값 집합으로 호출될 때마다 항상 동일한 결과를 반환하는 함수

※ non-deterministic function: Deterministic의 반대.입력 이외의 외부 상태(하드웨어 타이머, 난수 등)가 수행 과정에 적용되거나, 순서/시간에 민감하게 동작하여 nondeterministic 해질 수 있다.

※ 원자적 커밋: 여러 변경사항들을 하나의 운용 단위로 적용하는 것. 원자적 커밋이 끝나기 전에 실패한 것이 하나라도 있다면 원자적 커밋에서 완수되는 모든 변경사항들이 되돌려진다.

### 3.4 Locality

네트워크 대역폭은 상대적으로 컴퓨팅 환경에서 희소한 자원이다. MapReduce에서는 (GFS에 의해 관리되는)Input Data가 로컬 디스크에 저장된다는 점을 활용하여 대역폭을 절약한다. GFS는 Data를 64MB의 Block으로 나누어 Block의 복사본을 저장하는데, MapReduce의 Master는 복사본들의 위치 정보를 고려하여 Map Task가 데이터를 가지고 있는 Machine에서 실행되도록 스케쥴링한다. 만약 이것이 여의치 않을 경우, 최대한 복사본과 가까운 위치에서(ex. 같은 네트워크 스위치이 있는 machine) Map Task가 실행되도록 노력한다.

### 3.5 Task Granularity

우리는 보통 map phase를 M개로 나누고 reduce phase를 R개의 task로 나누는데, M과 R을 worker의 수보다 훨씬 많게 잡는 것이 이상적이다. worker가 많은 Task를 담당하면 Dynamic Load Balancing이 향상되고 worker에 장애가 발생해도 복구가 빠르다. (완료된 Map Task가 다른 worker로 퍼져 나갈 수 있기 때문.)

하지만 Master가 O(M+R)의 스케쥴링을 해야하고, O(M*R)의 메모리가 필요하기 때문에, M과 R의 현실적인 한계가 존재한다. 또, Reduce Task 개수만큼 Output 파일이 나오기 때문에 사용자가 R의 개수를 조정하기도 한다. 구글에서는 worker 2000대로 MapReduce로 돌릴 때 보통 M = 200000, R = 5000정도로 잡는다고 한다.

### 3.6 Backup Tasks

MapReduce에서 연산 시간을 증가시키는 보편적인 이유 중 하나는 straggler(유난히 map/reduce task 수행하는데 시간이 오래 걸리는 machine)이다. 연산이 다 끝나가는데 straggler에서 수행되지 못한 한 두개의 task 때문에 전체 소요 시간이 증가하게 된다.

MapReduce에서는 straggler가 일으키는 문제를 완화하기 위한 매커니즘이 있다. MapReduce가 거의 끝나갈쯔음이 되면, 남아있는 In-progress 상태의 Task Backup을 생성하고 스케쥴링하여, Primary / Backup Task 중 하나가 끝나게 되면 해당 Task를 완료된 것으로 표시한다. 이러한 튜닝은 전체 연산 시간을 극적으로 단축시킨다.

## 4 Refinements

몇몇 유용한 확장 기능 소개

### 4.1 Partitioning Function

보통 Map Task의 결과물은 중간 Key의 Hash를 사용한 default 파티셔닝 함수(ex. hash(key) mod R)로 파티셔닝 되는데, 보통 기본 파티셔닝 함수로도 Balancing이 잘되지만 때로는 Key의 다른 기능으로 파티셔닝을 하는 것이 더 좋을 때도 있다. 이럴 때 사용자는 Custom 파티셔닝 함수를 작성할 수 있다. 예를 들어, Key가 URL 값인데 최종 Output으로 Host 별로 File이 생성되길 원할 경우, Hash(Hostname(urlkey) mod R과 같은 함수를 사용하면 단일 Host의 모든 URL이  동일한 Output File에 모이도록 한다.

### 4.2 Ordering Guarantees

MapReduce는 중간 키/값 쌍이 Key 순서로 정렬되어 있음을 보장한다. 이는 사용자가 Data를 정렬해야하거나, Output file이 Random Access를 지원해야할때 편리하다.

### 4.3 Combiner Function

몇몇 경우에는 Map Task에서 반복적인 중간 키 값이 많이 발생하고, 사용자가 정의한 Reduce 함수가 교환법칙과 결합법칙을 만족할 수도 있다. (이를 만족하는 아주 좋은 예시는 Word Count이다.) 이 때, Combiner 함수를 작성하여 네트워크에 데이터를 전송하기 전에 부분적으로 데이터를 병합하여 전달할 수 있다. Combiner 함수는 Map Task가 실행되는 Machine에서 실행된다. 보통 Combiner 함수는 Reduce 함수와 동일한 코드로 구현되지만, Combiner함수의 결과는 중간 파일에, Reduce 함수의 결과는 최종 Output 파일에 작성된다는 차이가 있다.

![https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2017/09/mapreduce-program-with-combiner.jpg](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2017/09/mapreduce-program-with-combiner.jpg)

### 4.4 Input and Output Types

MapReduce는 input Data를 읽어들이기 위한 여러가지 포맷들을 지원한다. 예를 들어 Text Mode 입력은 각 줄을 키/값 쌍으로 처리하는데, 키에는 파일의 offset 정보가, 값에는 각 줄의 내용이 들어간다. 각 format은 Map Task에서 처리하기 적절한 형태로 가공될 수 있도록 구현되어 있다. 대부분은 제공되는 InputFormat 중 하나를 사용하지만, interface도 지원하기 때문에 사용자가 input type을 작성하여 사용할 수도 있다.

Input Format 참고: [https://data-flair.training/blogs/hadoop-inputformat/](https://data-flair.training/blogs/hadoop-inputformat/)

### 4.5 Side-effects

어떤 경우에는 Map/Reduce 연산으로부터 추가적인 정보를 담은 보조 파일을 생성하는 것이 유용했다. 이런 보조 파일과 관련된 연산이 원자적이고 멱등성(idempotent)을 만족하게 하는 것은 전적으로 사용자에게 의존한다.

※ Side-effect : [https://ko.wikipedia.org/wiki/부작용_(컴퓨터_과학)](https://ko.wikipedia.org/wiki/%EB%B6%80%EC%9E%91%EC%9A%A9_(%EC%BB%B4%ED%93%A8%ED%84%B0_%EA%B3%BC%ED%95%99))

### 4.6 Skipping Bad Records

Map이나 Reduce 함수에서 특정 레코드 작업 시 버그가 생기는 코드가 있을 수도 있다. 이럴 때에는 버그를 고치는게 일반적이지만, 데이터가 너무 많거나 버그를 일으키는 코드가 서드파티 라이브러리의 코드라면 고치지 못할 수도 있다. 이 경우에 MapReduce에서는 버그를 일으키는 레코드가 발견된 곳에서 해당 레코드를 스킵하는 모드를 옵션으로 제공한다. 이 모드에서는 특정 레코드에서 두 번 이상의 실패가 일어날 경우, 해당 레코드를 스킵하라고 표시한다. 그러면 레코드가 포함된 map/reduce task가 재실행 되었을 때 표시된 레코드가 바로 스킵된다.

### 4.7 Local Execution

MapReduce는 보통 수백수천개의 분산시스템 상에서 동작하고 Task도 Master에 의해 동적으로 할당되기 때문에, 사용자가 정의한 Map / Reduce 함수를 디버깅하는 건 까다로울 수 있다. 디버깅, 프로파일링, 소규모 테스팅을 용이하게 하기 위해 MapReduce는 작업을 로컬 머신에서 순차적으로 실행할 수 있게 하는 기능을 구현했다.

특정 Map Task만 실행되도록 제한할 수 있고, MapReduce 작업 실행 시 flag를 지정하여 실행하면 다른 외부 디버깅/테스팅 툴을 사용할 수 있다.

### 4.8 Status Information

Master는 내부에서 HTTP 서버를 실행하고 사용자가 활용할 수 있는 status page를 보낸다. 이를 통해 현재 작업 상황을 확인하고 얼만큼 시간이 더 걸리는지, 어디에서 버그가 발생하는지 등을 추적할 수 있다.

### 4.9 Counters

사용자는 처리된 총 단어수, 색인된 독일어 문서 수 등 특정 이벤트가 발생한 횟수를 계산하고 싶을 수 있다. MapReduce는 사용자가 name counter object를 만들어 횟수를 셀 수 있는 Counter라는 기능을 제공한다.

+) Map과 Reduce 연산이 잘 동작하는지 확인할때도 유용!

## 5 Performance / 6 Experience / 7 Related Work

5장에서는 문자열의 특정한 패턴을 찾는 grep과 정렬을 MapReduce로 실행해본 결과를 보여주고, 6장에서는 실제 구글에서 어떻게 MapReduce를 활용했는지 소개하고 있다. 7장은 MapReduce의 동작 방식과 관련하여 더 읽어볼만한 거리들을 소개하고 있는데, 셋 다 그냥 패스하도록 하겠다. ㅎㅎㅎㅎ

## 8 Conclusions

MapReduce는 구글에서 성공적으로 사용되고 있다. 그 이유를 꼽아보자면,

1. 병렬 / 분산 처리 경험이 없어도 사용할 수 있을만큼 쉽다.
2. 다양한 범위의 문제를 해결할 수 있다. (정렬부터 머신 러닝까지 ~)
3. 수천 대의 대규모 클러스터에서도 활용가능하다.

그리고 MapReduce를 개발하면서 저자들이 느낀 점이 있는데, 우선

1. **프로그래밍 모델을 제한**하면 연산을 병렬화하고 fault-tolerant를 만족하도록 하는 것이 쉽다.
2. 네트워크 대역폭은 희소한 자원이다.
    - 따라서 **네트워크를 타고 전송되는 데이터의 양이 최소화** 되도록 노력했고, 이 때문에 Map Task가 로컬 디스크에서 데이터를 읽어 들여서, 중간 결과물도 로컬 디스크에 저장하도록 구현했다.
3. 느린 기계의 영향력을 줄이고, 장애와 데이터 손실을 처리하는 데에 **중복 실행**을 사용할 수 있다.

논문 끗!

# Lecture Question

Q. map reduce는 여러 단계로 이루어져 있는지?

A. ㅇㅇ. map reduce의 결과가 다른 map reduce input으로 들어갈 수 있음.

Q. 함수 어떻게 수행?

A. master 서버는 특정 워커에게 map 실행 명령. 

Q. 아웃풋 어디로 가나?

A. 중간은 lcoal. reduce 결과는 cluster file system.

Q. GFS vs HDFS (조용래)

A. 비슷하지만 다르다. (김현우) GFS는 native(original)

Q. Athena가 뭐지

A. MIT 분산 처리 시스템

Q. reduce stream으로 가능?

A, mapreduce는 batch로 하지만 modern system에서 stream으로 하도록 노력하고 있음.

Q. map 작업 같은 노드에서 수행해야됨?

A. 아님. 

이제 네트워크 스펙이 좋아져서 예전처럼 map 작업을 굳이 데이터가 있는 노드에서 할 필요가 없어짐
따라서, 요즘 추세는 데이터노드와 그 데이터를 처리하는 노드 (실제로 spark 등이 돌아가는 노드)를 분리해서 많이 운영한다고 들었어요

정확히 프레스토가 이런 개념으로 메모리 대빵 좋은 노드로 클러스터 만들고, hdfs같은 데이터 소스에서 데이터를 모두 프레스토 클러스터로 읽어와서 처리합니다.

데이터 전용 노드에는 스토리지만 딥따 박아두고 데이터 처리 전용 노드에는 메모리만 딥다 박아서 클러스터 구성한다고 들었습니다.

아아 AWS EMR 도 보면 데이터노드랑 작업노드 같이있는 노드들을 scaling 할 수도 있고
task node라고해서 데이터노드와 붙어있지 않은 작업만 수행하는 노드들만 scaling 할 수 있어용!!

1. MapReduce client 프로그램이 입력 파일을 16 ~ 64MB 크기의 M조각으로 분할한 후, cluster에 복사
2. Master는 M개의 map task와 R개의 reduce task를 Worker에 할당
3. Map Task를 할당받은 worker는 input을 (key, value) 형태로 parsing하여 사용자가 정의한 map 함수로 전달한다. map 함수에 의해 생성된 중간 (key, value) 쌍은 메모리 버퍼에 보관
4. 버퍼에 담긴 데이터는 Partitioning Function (e.g.  *hash(key) mod R)*에 의해 R개의 영역으로 나뉘어 주기적으로 로컬 디스크에 저장되며,  위치는 master에게 전달되고 master는 이를 Reduce Task를 수행할 worker에게 다시 전달
5. Reduce Task를 할당받은 worker가 데이터가 저장된 로컬 디스크 위치를 전달받으면, RPC(Remote Procedure Call)을 이용하여 map worker에 있는 데이터를 읽고, data를 전부 읽으면 같은 key끼리 data를 그룹화할 수 있도록 key값을 기준으로 정렬 (데이터가 너무 크면 external sort를 활용)
6. Reduce worker는 모든 unique한 key와 그에 해당하는 value의 집합을 사용자가 정의한 reduce 함수에 전달하며, reduce함수의 결과값은 최종 output file에 추가
7. 모든 Map, Reduce Task가 끝나면 MapReduce 호출이 사용자 코드로 다시 돌아감
