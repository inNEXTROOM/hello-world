# Finding the Balance Between Speed & Accuracy During an Internet-wide Port Scanning(번역)

*포트 스캐닝 테스트에 대한 문서입니다.*

*오탈자 및 의역이 상당 수 존재할 수 있습니다.*

*원문은 https://captmeelo.com/pentest/2019/07/29/port-scanning.html 입니다.*

## Introduction

사전 탐색은 모든 버그 바운티 세션이나 침투 테스트 가장 중요한 단계라고 할 수 있다. 성공적인 사전 탐색은 성공과 실패 사이에서 차별점을 만들 수 있다. 사전 탐색은 능동적인 탐색과 수동적인 탐색 두 가지로 분류할 수 있다. 능동적인 사전 탐색의 방법중 하나가 포트 스캐닝이다. 이 방법을 통해 침투 테스터나 버그 헌터는 대상 호스트나 네트워크에 열려있는 포트를 확인하고 동작중인 서비스를 식별할 수 있다.

그치만.. 포트 스캐닝은 항상 속도와 정확도간의 상충이 생긴다. 침투 테스트를 이행하는 동안 테스터의 시간은 제한되어 있고, 버그 바운티는 버그를 가장 먼저 발견하고 제출하기 위해 항상 경쟁한다. 이러한 연유로 포트 스캐닝 중에는 정확도 보다는 속도를 우선시 한다. 단점은 이게 정확하지 않다는 것이다. 시간과의 싸움으로 성공적인 침투 테스트나 버그 헌팅을 이끌 수 있는 열린 포트를 놓칠 수가 있다.

이 연구는 오픈 소스와 잘 알려진 툴들을 사용해서 인터넷 상의 포트 스캐닝을 하는 동안의 속도와 정확도간의 균형을 찾는 것을 목적으로 한다.

## Port Scanning Overview

포트 스캐닝은 사전 탐색에서 가장 일반적으로 사용되는 테크닉 중 하나다. 이를 통해 침투 테스트나 버그 헌터는 호스트에서 사용 가능한 열린 포트와 실행되는 서비스를 식별할 수 있다.

포트 스캐너는 **연결형(동기식)** 과 **비 연결형(비동기식)** 스캐너로 구분할 수 있다.

### 연결형(동기식 방식)

이러한 종류의 스캐너는 목표 포트에 요청을 보내고 타임 아웃 기간까지 연결을 유지한다. 단점은 현재 연결이 닫히기 전까지 다음 대상의 IP나 포트로 진행하지 않기 때문에 느린 성능을 보인다는 것이다.

좋은 점은 손실되는 패킷까지 인식할 수 있기 때문에 정확도가 높다는 점이다. 

연결 형 스캐너 중에 제일 유명한 것으로는 **[Nmap](https://nmap.org/)** 이 있다.

### 비 연결형(비동기식 방식)

비 연결형 스캐너는 별도의 전송 및 수신 스레드를 가지기 때문에 현재 작업의 완료 여부가 다음 작업에 영향을 주지 않는다. 이 방법은 아주 빠른 속도의 스캐닝을 가능하게 하는데, 단점은 손실 패킷에 대한 처리가 없기 때문에 정확도가 더 낮다는 것이다.

**[Masscan](https://github.com/robertdavidgraham/masscan)** 과 **[Zmap](https://github.com/zmap/zmap)** 이 비 연결형 방식 중 가장 유명한 것들이다.

## Nmap vs Masscan

> 이 연구에서는 Nmap과 Masscan만 사용해서 진행한다. Zmap은 빠르고 결과물도 괜찮은 스캐너지만 한번에 하나의 포트만 가능하다. 경험상 Zmap의 스캐닝은 다중 작업을 동시에 실행한다고 해도 느리다.

Nmap과 Masscan이 제공하는 좋은 성능과 기능, 결과에도 불구하고, 여전히 고유의 약점을 가지고 있다. 아래의 표는 각 도구의 장, 단점이다.

> 두 도구간의 세부적인 비교가 아닌 점을 유의하세요! 연구에 관련된 점만 나열했습니다!

| | Nmap | Masscan |
|-|------|---------|
|장점 | 동기식 방식 사용으로 정확성이 더 높다. | 비동기식 방식을 사용해서 아주아주 빠르다.|
|     | 제공하는 기능이 많다.                 | 문법이 Nmap과 유사하다. |
|     |도메인 이름과 IP 주소(IPv4, IPv6 둘 다)를 쓸 수 있다.| |
|단점 | 수천 수백개의 대상을 스캐닝한다면 아주아주 느리다. | 빠른 속도로 큰 범위의 포트 스캐닝을 할 때 결과가 부정확하다.[1](https://github.com/robertdavidgraham/masscan/issues/365)|
|     |  |도메인 이름을 대상으로 쓸 수 없다.
|     |  |환경에 따라 자동적으로 속도를 조절하지 않는다.

## Research Idea

위 나열된 도구의 장단점을 기반으로 속도와 정확도 사이의 균형을 찾는 대에 있어 다음의 해결책과 문제점이 발견되었다.

### Solutions

도구의 장점을 기반으로 구성한 것은 다음과 같다.

1. Nmap의 정확도와 기능을 masscan의 속도와 결합한다.
2. Masscan을 사용하여 호스트에 오픈된 포트를 찾는 초기 포트 스캔을 수행한다.
3. Masscan의 결과를 입력 값으로 Nmap으로 디테일한 포트 스캐닝을 수행한다.

### Problems

위의 아이디어는 썩 괜찮지만, 여전히 단점을 해결해야한다. 구체적으로 다음과 같은 것들이 있다.

1. Nmap의 수천 수백개의 대상 스캐닝 할때의 아주 느린 속도
2. Masscan의 큰 범위 스캐닝의 부정확성 [[2]](https://github.com/robertdavidgraham/masscan/issues/365)

## Research Setup

### Target Networks

이번 연구를 위해 다음 네트워크의 서브넷이 선택되었음다.

|Target|Subnet|
|:----:|:----:|
|A|A.A.0.0/16|
|B|B.B.0.0/16|
|C|C.C.0.0/16|
|D|D.D.0.0.16|

### Scanning Machine

제한된 예산 때문에 하나의 스캐닝 머신만 사용하게 되었다. 사용된 장비는 DigitalOcean의 20불짜리 VPS로 4GB 램, 2 vCPU, 월 4TB 대역폭을 제공한다. 전체적인 연구에서 여기에 할당된 1개의 고정 IP 주소에섬만 진행했다. 

### Test Cases

이 연구에서 각 도구에서 사용가능한 옵션을 여러가지로 적용시킨 각자의 테스트 케이스를 가진다. 이 테스트는 도구의 단점을 해결하고 장점을 활용하여 속도와 정확도 간의 균형을 찾는 것을 목표로 한다.

**Masscan:**

1. 다양한 속도로 모든 TCP 포트를 일반(레귤러) 스캔한다.
2. `/16` 인 타겟 서브넷 대신 `/20` 을 사용하고, **X**개의 Masscan 동시작업을 각각 **Y**의 속도로 실행한다.
3. `1-65535` 포트 범위를 여러 범위로 분할하고, **X**개의 Masscan 동시작업을 각각 **Y**의 속도로 실행한다.

**Nmap:**

1. 모든 TCP 포트를 일반(레귤러) 스캔한다.
2. 모든 TCP 포트 스캐닝을 **X**개의 작업을 실행한다.
3. Masscan에 의해 식별된 오픈된 포트와 호스트의 결합된 목록을 스캔한다.
4. Masscan에 의해 식별된 특정 호스트의 특정 오픈 포트를 스캔한다.

> 제한된 시간 내 모든 옵션에 대한 내용을 다루는 것은 불가능하하여 이것만 했어요!

동시 작업을 사용하는 테스트 케이스에는 [GNU Parallel](https://www.gnu.org/software/parallel/)이 사용되었다. 처음 사용한다면 [튜토리얼](https://www.gnu.org/software/parallel/parallel_tutorial.html)을 확인해주셈.

## Scope and Limitations

- 연구에 사용된 도구는 `Nmap v7.70` 과 `Masscan v1.0.5-51-g6c15edc`
- IPv4 주소만 다룸
- UDP 포트 스캐닝은 포함하지 않음
- 가장 유명한 오픈 소스 도구만 사용함(*Zmap은 한번에 하나의 포트만 스캐닝할 수 있기 때문에, 다중 작업으로 실행한다고 해도 아주 느리다.*)
- 4개의 대상 네트워크만 탐색했으며 모두 `/16`
- 포트 스캐닝은 고정된 위지에 있는 하나의 IP를 사용한 1개의 머신에서만 수행
- Masscan 속도는 스캐닝 머신이 **RF_RING**을 지원하지 않아 **250kpps** 으로 제한
- 제한된 리소스로 인해 모든 테스트 케이스 조합이 수행되지 않음

## Masscan Test Cases & Results

이 섹션에서는 Masscan으로 수행 된 다양한 테스트 케이스 및 결과에 대해 자세히 설명한다.

### Test Case #1: Regular scan of all TCP ports with varying rates

이 테스트케이스에서는 특별할 것이 없다. 일반적인 Masscan을 이용한 스캐닝이지만 속도가 다르다.

이 테스트를 위해 아래의 명령어가 사용되었다.

    masscan -p 1-65535 --rate RATE --wait 0 --open TARGET_SUBNET -oG TARGET_SUBNET.gnmap

*Rates Used:*

- 1M
- 100K
- 50K

실험 중 VPS 에서 지원하는 속도는 250kpps 언저리이다. 스캐닝 머신이 **RF_RING**을 지원하지 않기 때문이다.

![](https://captmeelo.com/static/img/12/250kpps.jpg)

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test1.png)

*Observation:*

- Slower rate results into more open ports, but at the expense of a longer scanning time.
- 속도가 느려지면 오픈 포트를 많이 찾아내지만 스캐닝에 소요되는 시간이 길어진다.

### Test Case #2: Split the /16 target subnet into chunks of /20, and run X concurrent Masscan jobs, each with Y rate

동시 작업을 실행하기 위해서, `/16` 으로 나눈 서브넷 더 작게 쪼갰다. `/24` 처럼 더 작게 쪼갤수도 있겠지만 이번 연구에서는 `/20` 으로 사용하였다.

대상 네트워크의 서브넷을 더 작게 쪼개기 위해서, 아래의 파이썬 코드를 활용하였다.

    #!/usr/bin/python3
    import ipaddress, sys
    target = sys.argv[1]
    prefix = int(sys.argv[2])
    for subnet in ipaddress.ip_network(target).subnets(new_prefix=prefix): print(subnet)

위 코드의 스니펫은 다음과 같다.

![](https://captmeelo.com/static/img/12/split_sub.jpg)

각 작업에 사용되는 속도는 스캐닝 머신이 처리할 수 있는 최대치의 속도에 기반한다. 이 케이스의 경우 250kpps를 처리할 수 있기때문에 5개의 병렬 작업을 실행할 경우 50kpps의 속도를 사용할 수 있다.

> 머신의 속도 최대치는 절대적이지 않기때문에 각 작업의 속도를 최대 속도의 80-90%로 조절할 수 있다.

이번 테스트 케이스에서는 아래의 명령을 실행하였다. `[split.py](http://split.py)` 에서 생성된 더 작은 서브넷은 동시작업을 실행시키기 위한  `parallel` 의 입력 값으로 사용되었다.

    python3 split.py TARGET_SUBNET 20 | parallel -j JOBS "masscan -p 1-65535 --rate RATE --wait 0 --open {} -oG {//}.gnmap"

위의 명령을 실행했을 때의 결과는 아래와 같다. 이 경우에는 20개의 Masscan 작업이 각각 10kpps의 속도로 동시에 실행되었다.

![](https://captmeelo.com/static/img/12/masscan-20jobs.png)

*Rates and Jobs Used:*

- 5 jobs each w/ 100k rate
- 5 jobs each w/ 50k rate
- 20 jobs each w/ 10k rate

> 주의 사항:
- 스캐닝 머신 처리량인 250kpps로 진행했어야 하는데 잘못 계산해서 처음 속도와 작업 옵션을 5개 작업에 100k 속도로 설정해 총 속도가 500kpps이 되어버렸다. 그래도 결과는 아래 차트와 같이 여전히 가치가 있었다. 
- 20k 속도로 10개 동시 작업과 같은 조합도 가능하지만, 시간 및 예산 제한으로 모든 조합을 다룰 수는 없었다.

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test2.png)

*Observations:*

- 동시 작업으로 수행하면 일반 스캔(테스트 케이스 1)보다 2~3배 더 빠르지만, 오픈 포트의 수는 더 줄어든다.
- 스캐닝 기계의 최대 속도를 사용하면 열린 포트의 수가 줄어든다.(5개의 작업에 각 100k 속도)
- 동시작업을 낮추고 속도를 올리면 (5개의 작업에 각 50k 속도)가 동시작업을 높이고 속도를 낮추는 것(20개의 작업에 각 10k 속도)보다 낫다.

### Test Case #3: Split 1-65535 port range into several ranges, and run X concurrent Masscan jobs, each with Y rate

세번째 테스트 케이스는 `1-65535`와 같은 대규모 포트 범위를 지정할 때 발생하는 Masscan의 문제를 해결하려고 한다. 해결책은 `1-65535` 범위를 더 작게 나누는 것이다.

앞선 테스트 케이스와 같이, 작업과 속도는 최대 80~90%로 설정한다는 아이디어를 기반으로 하였다. 

아래의 명령이 이번 테스트 케이스에 사용되었다. `PORT_RANGES`는 포트 범위 목록을 포함하며 `parallel`에 대한 입력 값으로 사용된다. 

    cat PORT_RANGES | parallel -j JOBS "masscan -p {} --rate RATE --wait 0 --open TARGET_SUBNET -oG {}.gnmap"

`1-65535` 포트 범위를 4가지 방법(Split #1~4, 이하 스플릿)으로 나눴고 각각 작업과 속도에 대한 조합이 포함되어있다.

***Split #1:** 5 Port Ranges*

    1-13107
    13108-26214
    26215-39321
    39322-52428
    52429-65535

*Rates and Jobs Used:*

- 5 jobs each w/ 50k rate
- 2 jobs each w/ 100k rate

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test3-1.png)

***Split #2:** 2 Port Ranges*

    1-32767
    32768-65535

*Rates and Jobs Used:*

- 2 jobs each w/ 100k rate
- 2 jobs each w/ 125k rate

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test3-2.png)

***Split #3:** 8 Port Ranges*

    1-8190
    8191-16382
    16383-24574
    24575-32766
    32767-40958
    40959-49151
    49152-57343
    57344-65535

*Rates and Jobs Used:*

- 4 jobs each w/ 50k rate
- 2 jobs each w/ 100k rate

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test3-3.png)

***Split #4:** 4 Port Ranges*

    1-16383
    16384-32767
    32768-49151
    49152-65535

*Rate and Job Used:*

- 2 jobs each w/ 100k rate

> 4 포트 분할에 하나의 작업 속도 조합만 사용한 이유는 월 대역폭을 다 소진했기 때문입니다.(눈물) 그래서 추가 지출이 있었음.

*Charts:*

![](https://captmeelo.com/static/img/12/masscan-test3-4.png)

*Observations:*

> 아래의 결과는 4개의 스플릿을 포함한 것이다.

- 포트 범위를 분할하면 더 많은 오픈 포트가 결과에 포함된다.
- 더 적은 다중 작업을 사용했을 때 더 많은 오픈 포트가 결과에 포함된다.
- 테스트한 스플릿 중에서 가장 베스트는 스플릿 #1이었다.

### Raw Data

다음 표는 위의 Masscan 테스트 케이스에서 수집된 원시 데이터이다.

![](https://captmeelo.com/static/img/12/masscan-raw.jpg)

### Masscan Conclusion

Masscan으로 측정한 테스트 케이스를 바탕으로 다음과 같은 결과를 도출 하였다.

- CPU를 100%로 굴리면 오픈 포트 결과 값이 줄어든다.
- 머신이 지원하는 최대 속도를 사용할 경우에도 오픈 포트 결과 값이 줄어든다.
- 병렬 작업으로 진행했을 때, 작업 수를 줄이는 것이 더 많은 포트를 찾았다.
- 포트 범위를 나누는 것이 서브넷을 나누는 것보다 나은 결과를 보였다.
- 포트 범위가 4~5개로 분할 될 때 최선의 결과를 얻을 수 있었다.

## Nmap Test Cases & Results

이 단계에서는, 버전 스캐닝만 포함했으며 Nmap의 NSEs, OS 추측 및 다른 스캐닝 기능은 포함하지 않았다. Nmap 스레드는 `T4` 옵션과 같은 다음과 같은 속도로 지정이 되어있다.

    --max-rtt-timeout=1250ms --min-rtt-timeout=100ms --initial-rtt-timeout=500ms --max-retries=6 --max-scan-delay=10ms

Masscan이 사용하는 옵션과 비슷하게 하기위해 다음 옵션이 사용되었다. 이 옵션은 모든 테스트 케이스에 적용시켰다.

*Options Used:*

- SYN scan (`-sS`)
- Version scan (`-sV`)
- Threads (`-T4`)
- Randomize target hosts order (`--randomize-hosts`)
- No ping (`-Pn`)
- No DNS resolution (`-n`)

### Test Case #1: Regular scan of all TCP ports

이 테스트 케이스는 Nmap을 사용한 노멀 스캔으로 딱히 특별할 것은 없다.

사용한 커맨드는 다음과 같다.

    sudo nmap -sSV -p- -v --open -Pn -n --randomize-hosts -T4 TARGET_SUBNET -oA OUTPUT

*Observations:*

- 4.5일 이후에도 작업이 완료되지 않았다(헐). 이것은 상기에서 언급한 단점 중 하나로 큰 네트워크를 대상으로 스캐닝을 할 때 Nmap은 아주 느리다.
- 매우 느린 속도로 테스트 케이스를 취소하기로 결정했다.

### Test Case #2: Scan of all TCP ports using X concurrent jobs

이 테스트 케이스에서는 Nmap 스캐닝을 동시 작업으로 진행시켜 느린 퍼포먼스를 해결하고자 했다. 이 테스트는 위의 Masscan 케이스와 마찬가지로 서브넷을 더 작은 단위로 쪼개서 수행했다. 또다시 아래의 파이썬 코드 `split.py` 를 이용해 대상 서브넷을 나누어 주었다. 

    #!/usr/bin/python3
    import ipaddress, sys
    target = sys.argv[1]
    prefix = int(sys.argv[2])
    for subnet in ipaddress.ip_network(target).subnets(new_prefix=prefix): print(subnet)

테스트 케이스에 쓰인 명령은 다음과 같다.

    python3 split.py TARGET_SUBNET 20 | parallel -j JOBS "sudo nmap -sSV -p- -v --open -Pn -n --randomize-hosts -T4 {} -oA {//}"

이 테스트 케이스에서는 아래와 같이 두 개의 병렬 작업 인스턴스를 사용하기로 결정했다.

***5개 동시작업:** `/16` 서브넷을 `/20` 서브넷으로*

*Observation:*

- 아주 느렸다. 2.8일 후에도 작업이 완료되지 않아 작업을 취소했다.

***64개 동시작업:** `/16` 서브넷을 `/24` 서브넷으로*

*Observation:*

- 5일이 지나도 끝나지 않아서 이것 역시 취소했다.

### Test Case #3: Scan on the combined list of open ports and hosts identified by Masscan

이번 테스트 케이스의 기본 개념은 먼저 Masscan에 의해 탐지된 호스트 목록과 오픈 포트의 목록을 얻는 것이다. Nmap 테스트  케이스에서 더 많거나 적은 오픈 포트를 탐지할 수 있는지에 대해 오픈 포트 목록(아래 차트에서의 녹색 막대)을 기준으로 사용했다.

예를 들어, Masscan이 300개의 오픈 포트를 탐지하고 Nmap의 일반 스캔이 320개의 오픈 포트를 탐지했다. 하지만 5개의 공동 작업으로 Nmap 스캐닝을 했을 때는 단지 295개의 오픈 포트를 찾았다. 이것은 일반 스캔으로 진행하는것이 더 낫다는 것을 의미한다.

Masscan의 결과로부터 호스트의 목록을 얻기 위해 다음 명령을 사용했다.

    grep "Host:" MASSCAN_OUTPUT.gnmap | cut -d " " -f2 | sort -V | uniq > HOSTS

위 명령은 아래와 같은 결과를 보여준다.

![](https://captmeelo.com/static/img/12/nmap-test3-hosts.jpg)

아래는 Masscan의 모든 열린 포트를 받아오는데 사용한 명령이다.

    grep "Ports:" MASSCAN_OUTPUT.gnmap | cut -d " " -f4 | cut -d "/" -f1 | sort -n | uniq | paste -sd, > OPEN_PORTS

결과는 다음과 같다.

![](https://captmeelo.com/static/img/12/nmap-test3-ports.jpg)

이 명령은 Nmap 일반 스캔에 사용되는 명령이다.

    sudo nmap -sSV -p OPEN_PORTS -v --open -Pn -n --randomize-hosts -T4 -iL HOSTS -oA OUTPUT

다음은 Nmap 스캐닝 동시 작업에 사용된 명령이다. 위에서 생성한 호스트와 포트 정보를 사용한다.

    cat HOSTS | parallel -j JOBS "sudo nmap -sSV -p OPEN_PORTS -v --open -Pn -n --randomize-hosts -T4 {} -oA {}"

*Jobs Used:*

- 0 (this is a regular Nmap scan)
- 10
- 50
- 100

*Charts:*

![](https://captmeelo.com/static/img/12/nmap-test3.png)

*Observations:*

- 일반 스캔을 수행했을 때 CPU 사용량은 고작 10% 근처였다.
- 일반 Nmap 스캐닝이 다중 작업으로 진행했을 때보다 더 많은 오픈 포트를 찾아냈다.
- (차트의 녹색 막대) 기준점과 비교해서, 서브넷 A는 더 많이 탐지가 되었지만 서브넷 B와 C에 비하면 적은 숫자만 식별된 것을 확인할 수 있었고 서브넷 D의 경우 차이점을 보이지 않았다.

**The Additional Open Ports Detected by Nmap**

아래 표를 보자. Masscan이 각 호스트에서 다음과 같은 포트(열 2)를 찾았다고 가정해보자. Masscan이 탐지한 모든 열린 포트는 Nmap 스캔(열 3)이 실행할 때 대상 포트로 사용된다.

이 예제에서 Nmap에 의해 새로운 오픈 포트가 탐지되었다(열 4의 볼드체). 어떻게 이런 일이 일어났을까? Masscan은 비동기식 스캐너이기 때문에 `192.168.1.2`와 `192.168.1.3` 호스트에서 `22`번 포트가 누락이 되었을 수도 있다. 모든 호스트에서 탐지된 모든 오픈 포트를 합쳐서 Nmap의 대상 포트에 사용했으므로 누락된 포트(`22`)가 다시 탐색 될 있을 것이다. 주의점은 Nmap 스캐닝에 영향을 줄 수 있는 다른 요소가 있기 때문에 Nmap이 이를 감지할 수 있을 것이라는 것에 대한 보장이 없다.

[](https://www.notion.so/03111205f4c847d4a078f9a14fb7f520#655a8100e1f8425abeae4d3294101f2b)

### Test Case #4: Scan on the specific open ports on specific hosts identified by Masscan

이번에는 이전 테스트 케이스와 유사하다. 여기서는 Masscan이 감지한 모든 호스트와 오픈 포트를 결합하지 않는다. 특정 호스트에서 Masscan이 감지한 오픈 포트가 무엇이든 Nmap은 동일한 포트를 대상으로 수행할 것이다. 다음의 표는 이 테스트 케이스에서 수행 된 작업을 보여준다.

[](https://www.notion.so/03111205f4c847d4a078f9a14fb7f520#611ceda6fe144b6f8a1a21ceee5ad354)

아래의 명령이 호스트 목록을 얻기 위해 사용되었다.

    cat MASSCAN_OUTPUT.gnmap | grep Host | awk '{print $2,$5}' | sed 's@/.*@@' | sort -t' ' -n -k2 | awk -F' ' -v OFS=' ' '{x=$1;$1="";a=a","$0}END{for(x in a) print x,a}' | sed 's/, /,/g' | sed 's/ ,/ /' | sort -V -k1 | cut -d " " -f1 > HOSTS

그림은 명령이 실행 되었을 때를 보여준다.

![](https://captmeelo.com/static/img/12/nmap-test4-hosts.jpg)

각 호스트에서 열린 포트 목록을 가져오기 위해 실행한 명령이다.

    cat MASSCAN_OUTPUT.gnmap | grep Host | awk '{print $2,$5}' | sed 's@/.*@@' | sort -t' ' -n -k2 | awk -F' ' -v OFS=' ' '{x=$1;$1="";a=a","$0}END{for(x in a) print x,a}' | sed 's/, /,/g' | sed 's/ ,/ /' | sort -V -k1 | cut -d " " -f2 > OPEN_PORTS

명령 실행 결과는 다음과 같다.

![](https://captmeelo.com/static/img/12/nmap-test4-ports.jpg)

보시다시피 출력 결과가 테스트 케이스 #3과는 다르다. 모든 오픈 포트를 결합하는 대신 각 호스트로부터 찾은 모든 오픈 포트 목록을 만들었다.

`::::` 옵션션을 사용하여 두 목록을 `parallel` 입력 값으로 사용하고 Nmap 스캐닝을  동시에 실행했다.

> 다시 한번 말하는데, GNU Parallel에 익숙하지 않으면 튜토리얼 보셈

    parallel -j JOBS --link "sudo nmap -sSV -p {2} -v --open -Pn -n -T4 {1} -oA {1}" :::: HOSTS :::: OPEN_PORTS

위의 두 이미지를 기반으로 `parallel` 명령을 사용해서 동시 스캐닝을 실행했을 때 발생하는 예다.

    sudo nmap -sSV -p 443 -v --open -Pn -n -T4 192.168.1.2 -oA 192.168.1.2
    sudo nmap -sSV -p 80,443,1935,9443 -v --open -Pn -n -T4 192.168.1.5 -oA 192.168.1.5
    sudo nmap -sSV -p 80 -v --open -Pn -n -T4 192.168.1.6 -oA 192.168.1.6
    sudo nmap -sSV -p 80,443 -v --open -Pn -n -T4 192.168.1.7 -oA 192.168.1.7
    sudo nmap -sSV -p 08,443 -v --open -Pn -n -T4 192.168.1.9 -oA 192.168.1.9

다음 이미지는 테스트 케이스가 실행되었을 때 나타나는 스니펫이다. 아래에서 볼 수 있듯이 10개의 동시 작업을 `parrallel`을 사용해서 진행했다.

![](https://captmeelo.com/static/img/12/nmap-test4-1.jpg)

*Jobs Used:*

- 10
- 50
- 100

*Charts:*

![](https://captmeelo.com/static/img/12/nmap-test4-2.png)

*Observations:*

- 더 많은 동시작업과 머신이 100% CPU 사용량으로 동작하면 오픈 포트가 줄어든다.
- 10~50개의 병렬 스캔 결과의 차이는 크지 않으므로 시간을 줄이기 위해서는 50개의 병렬 작업을 수행하는 것이 낫다.
- 이번 테스트 케이스는 테스트 케이스 #3보다 빠르지만 오픈 포트 발견은 더 적다.

### Raw Data

다음 표는 위에서 언급한 Nmap 테스트에 대한 원시 데이터이다.

![](https://captmeelo.com/static/img/12/nmap-raw.jpg)

### Nmap Conclusion

Nmap으로 수행한 실험 결과를 통해 다음과 같은 결과를 도출해 내었다:

- 테스트 케이스 #3과 같이 Masscan에 의해 식별된 오픈 포트를 결합하여 Nmap 스캐닝을 수행하면 최상의 결과를 도출 할 수 있다. 또한 추가적인 오픈 포트를 찾을 수 있기 때문에 권장되는 방법이다.
- CPU 사용률이 100%이라면 오픈 포트는 줄어든다.
- 병렬 작업을 수행하는 경우 적은 병렬 작업 수가 더 많은 오픈 포트를 도출해낸다.

# Research Conclusion

### Recommended Approach

Masscan과 Nmap에 대해 수행된 테스트 케이스 결과를 바탕으로 인터넷 포트 스캐닝을 하는 동안 속도와 정확도를 위해 다음과 같은 방법이 권장된다.

1. 2,3개의 Masscan 동시작업을 실행하고 모든 65535개의 포트를 4,5 단위로 분할한다.
2. Masscan의 결과에서 호스트 목록과 오픈 포트 목록을 가져온다.
3. 이 목록을 Nmap에 대한 입력 값으로 사용하되 일반 스캔을 사용한다.

### Precautions

두 도구 모두 다음과 같은 예방 조치를 취해 오픈 포트가 줄어들지 않도록 한다.

- CPU 과부하를 피한다.
- 스캐닝 머신의 최대 속도를 사용하지 않는다.
- 너무 많은 병렬 작업을 수행하지 않는다.

# Final Thoughts

이 연구는 인터넷 포트 스캐닝을 하는 동안 속도와 정확도 사이의 균형을 유지하는 방법을 제공하지만, 100% 신뢰할 수 있는 결론으로 여기면 안된다. 제한된 시간과 예산으로 몇 가지 요인이 연구에 포함되지 않았다. 특히, 전체 과정에서 하나의 IP 주소만 사용하는 것은 좋은 설정이 아니었다. 동일한 대상 네트워크를 여러번 스캔했기 때문에 스캐닝 머신의 IP 주소가 블랙리스트에 올라가 오픈 포트의 수가 정확하게 나오지 않았을 수 있다. **Scope and Limitations** 섹션에서 재논의해 연구 결과에 영향을 줄 수 있는 몇가지 요소에 대해 제공해주세용.

# Closing

그럼 이만.
