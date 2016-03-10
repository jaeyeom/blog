# Go 언어로 우버에서 초당 요청수가 가장 많은 서비스를 어떻게 개발했나? #

저자: Kai Wei / 번역: 염재현
2016년 2월 24일 작성 / 2016년 3월 8일 번역

원문주소: https://eng.uber.com/go-geofence/

*이 글은 원저자인 Kai Wei씨의 허락 하에 우버 블로그 포스트인 "HOW WE BUILT UBER ENGINEERING’S HIGHEST QUERY PER SECOND SERVICE USING GO"를 번역한 것입니다.*

![enter image description here](https://eng.uber.com/wp-content/uploads/2016/02/WP_20140823_09_20_45_Pro__highres-edit_small.jpg)

2015년 초에 저희는 한 가지만 하는 (그리고 그 한 가지를 굉장히 잘 하는) geofence 조회 서비스를 만들었습니다. 한 해가 지나고, 이 서비스는 우버에서 서비스하는 수백개의 서비스 중에서 초당 조회수(QPS)가 가장 높은 서비스가 되었습니다. 왜 저희가 이 서비스를 만들었는지, 그리고 비교적 최근에 태어난 [Go 프로그래밍 언어](https://golang.org/)로 어떻게 이렇게 빨리 서비스를 만들고 크게 키울 수 있었는지에 대하여 이야기하려고 합니다.

## 배경 ##

우버에서 *geofence*는 지구 표면 위에 있는 사람이 정의한(human-defined) 지리적 구역(기하학적 용어로는 다각형)을 뜻합니다. geofence는 우버에서 지리 기반 설정에 광범위하게 사용됩니다. 특정 위치에서 어떤 상품을 이용할 수 있는지를 보여주고, 공항 같은 특별 요구사항이 필요한 지역을 정의하고, 많은 사람들이 동시에 탑승 요청을 하는 동네에서 탄력적인 가격 조정을 구현하기 위하여, 이 서비스는 매우 중요합니다.

![콜로라도에 있는 geofence의 예시](https://eng.uber.com/wp-content/uploads/2016/02/geofence-example-1024x796.png)
*콜로라도에 있는 geofence의 예시*

사용자의 모바일 폰의 [위도-경도](https://ko.wikipedia.org/wiki/%EC%A7%80%EB%A6%AC_%EC%A2%8C%ED%91%9C%EA%B3%84) 위치에 따른 설정값을 구하기 위한 첫 번째 단계는 해당 위치가 어떤 geofence에 들어가는지를 찾는 것입니다. 이 기능은 여러 서비스와 모듈에 분산되고 중복되어 있었습니다. 그러나 [단일체 구조에서 (마이크로)서비스 기반의 구조로 이전](https://eng.uber.com/soa/)하면서 이 기능을 새 마이크로서비스 하나에 중앙 집중시켰습니다.

## 제자리에, 차려, 출발! (Ready, Set, Go!) ##

언어들을 평가하고 있을 무렵, 실시간 장터(real-time marketplace) 팀에서 주로 쓰는 프로그래밍 언어가 [Node.js](https://ko.wikipedia.org/wiki/Node.js)였으므로 사내에서는 Node.js에 대하여 더 많은 지식이 있었습니다. 그러나 다음과 같은 이유로 Go 언어가 더 적합했습니다!

 - **높은 처리량과 낮은 지연 시간.** geofence 조회 서비스는 굉장히 많은(초당 수십만 개) 요청을 하나하나 모두 짧은 시간(백분위로 99%의 요청을 100밀리초 이내)에 처리해야 합니다.
 - **CPU에 집중적으로 걸리는 로드.** geofence 조회 서비스는 CPU가 많이 필요한 [다각형 내에 있는 점 판별](https://en.wikipedia.org/wiki/Point_in_polygon) 알고리즘을 풀어야 합니다. Node.js는 입출력(I/O)이 집중적으로 필요한 우버 내의 다른 서비스에는 적합하지만, Node.js의 [인터프리터 방식](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%84%B0%ED%94%84%EB%A6%AC%ED%8A%B8_%EC%96%B8%EC%96%B4)과 [동적 정형](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%A3%8C%ED%98%95_%EC%B2%B4%EA%B3%84#.EB.8F.99.EC.A0.81_.EC.A0.95.ED.98.95) 때문에 이 경우에 적합하지 않습니다.
 - **백그라운드 로딩의 비방해성(non-disruptive).** 최신의 geofence 자료가 활용되는 것을 보장하려면, 이 서비스는 메모리 상에 있는 geofence 자료를 여러 자료원으로부터 계속해서 백그라운드에서 갱신해 주어야 합니다. Node.js가 [단일 쓰레드](https://en.wikipedia.org/wiki/Thread_%28computing%29#Single_threading) 구조이기 때문에, 백그라운드 갱신 중에 CPU를 장기간(예를 들어 CPU가 많이 필요한 [JSON](https://ko.wikipedia.org/wiki/JSON) 파싱 같은 작업이 수행되는 기간 동안)에 걸쳐서 붙잡고 있어야 하고, 이것 때문에 조회 요청에 대한 응답 시간이 튀는 현상이 발생합니다. Go 언어에서는 [고루틴](https://gobyexample.com/goroutines)들이 여러 CPU 코어에서 실행되어 백그라운드 작업들이 포그라운드의 조회 작업들과 함께 병렬적으로 수행될 수 있기 때문에, 이것이 문제가 되지 않습니다.

### 위치 색인을 하느냐 마느냐: 그것이 문제로다 ###

위도-경도 위치가 주어졌을 때, 수만 개의 geofence들 중에서 이것이 어디에 속하는지는 어떻게 찾을 수 있을까요? 무차별 대입(brute-force) 방법은 간단합니다. 모든 geofence에 대하여 다각형 안에 점이 들어가는지를 [광선 쏘기 알고리즘](https://en.wikipedia.org/wiki/Point_in_polygon#Ray_casting_algorithm) 등으로 검사하면 됩니다. 그러나 이 방법은 너무 느립니다. 어떻게 하면 효율적으로 탐색 공간을 줄일 수 있을까요?

[R 트리](https://ko.wikipedia.org/wiki/R_%ED%8A%B8%EB%A6%AC)나 난해한 [S2](http://blog.christianperone.com/2015/08/googles-s2-geometry-on-the-sphere-cells-and-hilbert-curve/)로 geofence들을 색인하는 대신에, 저희들은 우버의 사업 모델이 도시 중심적이라는 것에 착안하여 더 단순한 길을 선택했습니다. 사업 규칙(비지니스 룰)이나 그것을 정의하는 geofence는 주로 도시와 연관되어 있습니다. 그래서 도시의 geofence(도시 경계를 정의하는 geofence)를 첫 번째 단계로 두고, 각각의 도시 안에 있는 geofence를 두 번째 단계로 두는 2단계 계층으로 geofence들을 구성했습니다.

각각의 조회 요청에 대하여 선형 탐색(linear scan)으로 먼저 도시의 geofence를 찾고, 그 도시 안에 들어 있는 geofence들을 또 다른 선형 탐색으로 찾습니다. 이 해법의 실행 시간 복잡도는 [O(N)](https://ko.wikipedia.org/wiki/%EC%A0%90%EA%B7%BC_%ED%91%9C%EA%B8%B0%EB%B2%95)으로 동일하지만, 이런 단순한 테크닉으로 N이 만 단위에서 백 단위로 줄어들었습니다.

## 설계 ##

저희는 이 서비스를 상태 없이(stateless) 만들어서 각각의 요청을 아무 서비스 인스턴스에서 받아서 처리해도 같은 결과가 나오게 하고 싶었습니다. 이것은 정보를 분할해서 갖고 있는 것이 아니라 인스턴스 하나하나가 지구 전체의 정보를 다 갖고 있어야 한다는 뜻입니다. [결정론적인(deterministic)](https://en.wikipedia.org/wiki/Deterministic_system) 폴링 스케쥴을 생성해서 geofence 자료가 다른 서비스 인스턴스들과 동기화 되도록 만들었습니다. 따라서 이 서비스는 설계가 아주 단순합니다. 백그라운드 작업이 주기적으로 geofence 자료를 여러 자료 저장소에서 퍼옵니다. 그 다음 이 자료는 주 기억장치에 저장되어서 조회 요청에 응답하고, 서비스가 재시작했을 때 빨리 일어설 수 있도록 로컬 파일 시스템으로 직렬화됩니다.

![geofence 조회 서비스 설계](https://eng.uber.com/wp-content/uploads/2016/02/go-geofence-service-architecture-1024x621.png)
*geofence 조회 서비스 설계*

### Go 메모리 모델 다루기 ###

이 설계를 구현하려면 메모리에 있는 지리 색인에 병행적으로 읽기/쓰기 접근이 필요합니다. 특히 백그라운드 폴링 작업은 포그라운드 조회 엔진이 색인에서 읽어오는 동안에 메모리에 씁(write)니다. 단일 쓰레드 기반의 Node.js 세상에서 온 사람들에게 [Go 메모리 모델](https://golang.org/ref/mem)은 어렵게 느껴질 수 있었습니다. 고루틴과 [채널](https://gobyexample.com/channels)로 병행적으로 읽기/쓰기를 동기화 하는 것이 Go의 특징인데, 이렇게 하면 성능에 부정적인 영향이 있을 것이라는 우려가 있었습니다. [sync/atomic](https://golang.org/pkg/sync/atomic/) 패키지에 있는 *StorePointer/LoadPointer* 같은 저수준 기능을 활용하여 메모리 장벽을 스스로 관리하고자 했으나, 코드가 쉽게 깨지고 유지보수하기 어려워졌습니다.

결국, [읽기/쓰기 잠금(RWLock)](https://golang.org/pkg/sync/#RWMutex)을 이용하여 지리 색인으로의 접근을 동기화하는 방식으로 결정했습니다. 잠금(lock)이 서로 충돌하는 것을 최소화하기 위하여, 새로 들어오는 색인 조각들은 따로 만들어진 다음에 주 색인과 원자적으로(atomically) 교환되었습니다. 이렇게 잠금을 사용하니 *StorePointer/LoadPointer*를 이용했을 때보다 조금 지연 시간이 증가하였지만, 단순하고 유지 보수 가능한 코드 베이스를 위한 약간의 성능 비용은 충분히 낼 가치가 있다고 믿습니다.

## 우리의 경험 ##

뒤돌아 보자면, 저희가 [Go](https://ko.wikipedia.org/wiki/Go_%28%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D_%EC%96%B8%EC%96%B4%29)를 선택하여(Go for it) 새로운 언어로 서비스를 만든 것은 매우 만족스러운 결정이었습니다. 강조드릴 점은:

 - **높은 개발자 생산성.** Go 언어는 C++, Java, Node.js 개발자가 배우는데 그저 며칠 정도 밖에 걸리지 않고, 코드를 쉽게 유지 보수할 수 있습니다. ([정적 정형](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%A3%8C%ED%98%95_%EC%B2%B4%EA%B3%84#.EC.A0.95.EC.A0.81_.EC.A0.95.ED.98.95) 덕분에 더 이상 추측을 해야 할 필요도 없고, 불쾌하게 깜짝 놀랄 일도 없습니다.)
 - **높은 처리량과 낮은 지연 시간.** 중국을 제외한 모든 나라에 서비스되는 저희 주 데이터 센터는 2015년말 기준으로 40대의 기계가 35% CPU 사용률을 유지하면서 170k QPS(초당 17만 조회)를 처리하고 있습니다. 백분위로 95%의 요청들이 모두 5밀리초 이내로 처리되고, 99%의 요청들이 모두 50밀리초 이내로 처리됩니다.
 - **초특급 신뢰성.** 이 서비스는 시작된 이후로 99.99%의 가동률을 보이고 있습니다. 유일한 가동중지가 초보적인 프로그래밍 실수와 써드파티 라이브러리에 있던 파일 서술자(file descriptor) 누출 버그로부터 발생했습니다. 중요한 점은, Go 런타임에서는 어떤 문제도 발견할 수 없었다는 것입니다.

## 앞으로 가야(Go)할 길 ##

지금까지 우버는 Node.js와 파이썬을 주로 이용해 왔지만, 우버의 새 서비스 중 다수에 Go 언어가 채택되고 있습니다. 우버에서 Go의 기세가 대단하기 때문에, Go 언어의 전문가나 아니면 초보자로 열정이 있으시다면, 저희는 Go 개발자를 [채용](https://www.uber.com/careers/list/?city=all&amp;country=all&amp;keywords=&amp;subteam=all&amp;team=engineering)하고 있습니다. [오, 당신이 갈(Go) 그곳들! (Oh, the places you'll Go!)](https://en.wikipedia.org/wiki/Oh,_the_Places_You%27ll_Go!)

사진 출처: "Golden Gate Gopher" by Conor Myhrvold, Golden Gate Park, San Francisco.

헤더 설명: [고퍼(The Go Gopher)](https://blog.golang.org/gopher)는 "아이코닉한 마스코트이자 가장 독특한 Go 프로젝트의 특징"으로 표현된다.
