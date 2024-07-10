
# [김서영] spark local 작업 환경 구축

## Introduction

해당 글은 [spark 로컬 작업 환경 레포](https://github.com/brickstudy/infra-docs/tree/main/spark) 를 안내하는 글입니다. 해당 작업 환경을 구성하는데에 고려한 배경 지식을 간단히 정리하여 같은 context를 공유하고, 개발한 과정과 결과를 소개하여 발전 방향 및 피드백을 수렴하고자 합니다.


## Table of Contents

1. [Introduction](#introduction)
2. [Background](#background)
    1. [Spark?](###🐣1.-Spark?)
    2. [Why Spark? ](###🐥2.-why-spark?)
    3. [pandas vs spark](###👯‍♀️3.-pandas-vs-spark)
    4. [Spark component](###👩‍👧‍👦-4.-Spark-Component)
    5. [Spark execution modes](###🏋️‍♀️-5.-Spark-Execution-Modes)
3. [Main Content](#main-content)
    1. [main development environment](###main-development-environment)
    2. [개발 환경 분리](###개발-환경-분리)
    3. [guide](###guide)
    4. [etcs, further](###Etcs,-Further)

<br>

## Background
 
### 🐣1. Spark?
Apache Spark는 통합 컴퓨팅 엔진압나다. 클러스터 환경에서 데이터를 병렬로 처리하는 라이브러리 집합이고, 현재 가장 활발하게 개발되고 있는 병렬 처리 엔진입니다.
- 가장 널리 쓰이는 네 가지 언어(scala, java, python, r) 지원
- sql, streaming, ml에 이르는 라이브러리 제공, 단일 노트북 환경부터 수천 대 서버로 구성된 클러스터까지 다양한 환경에서 실행될 수 있기 때문에, 빅데이터 처리를 쉽게 시작해서 upper bound 없는 큰 규모 클러스터로 확장해나갈 수 있음

> cluster computing <br> 
> 보통 컴퓨터라 함은, 책상 위에 놓여져있는 장비 한 대를 떠올립니다. 이 한 대 컴퓨터로는 연산할 수 없는 데이터 처리 작업이 있는데, 컴퓨터 클러스터는 여러 컴퓨터의 자원을 모아 하나의 컴퓨터처럼 사용할 수 있게 만들어 이를 해결합니다. 하지만 컴퓨터 클러스터를 구성하는 것만으로는 부족하고, 클러스터에서 작업을 조율할 수 있는 프레임워크가 필요합니다. 스파크는 이 역할을 수행합니다. 클러스터 내에서 데이터 처리 작업을 관리하고 조율해주는 것입니다.

>computing engine<br>
>스파크는 computing engine입니다. 영구 저장소의 역할을 수행하지 않고, 저장소 시스템의 데이터를 연산하는 역할만 수행합니다. 대신 다양한 저장소를 지원하여 데이터 저장 위치에 상관없이 처리에 집중합니다.

### 🐥2. why spark?
- Fast Processing - Resilient Distributed Dataset (RDD)는 immutable한 분산 객체 집합입니다. RDD에 있는 각 데이터셋들은 논리적인 partition들로 나눠져있고, 이 단위로 클러스터의 노드들에 분산되고, 연산이 이루어집니다.
- In-memory computing - 데이터가 RAM에 저장되어 (cached) 데이터 분석 속도가 뻐릅니다.
- Flexibility - 여러 언어를 지원합니다. (Scala, Python, R, Java)
- Fault tolerance - RDD를 통해 데이터 신뢰성 및 복구 가능성을 보장합니다.
- Analytics - Advanced Analytics, Machine Learning, deep learning과 같은 advanced 분석이 built-in MLlib를 통해 가능합니다.


### 👯‍♀️3. pandas vs spark

분산 처리가 되는지, 되지 않는지의 차이!

pandas는 파이썬 라이브러리로 dataframe 객체가 분산 컴퓨터가 아닌 단일 컴퓨터에 존재하지만, spark Datafrmae은 spark에서 제공하는 대표적인 구조적 api로, 여러 컴퓨터에 분산되어있습니다.

|Spark DataFrame|Pandas DataFrame|
|:---:|:---:|
|parallelization 지원|parallelization 지원하지 않음|
|여러 노드에 걸쳐 실행|단일 노드에서 실행|
|Lazy Evaluation 따름|Eager Execution 따름|
|immutable|mutable|
|상대적으로 복잡한 operation을 수행하기 어려움|상대적으로 수월함|
|대규모 데이터에서 처리 속도가 빠름|대규모 데이터 처리 속도가 더 느림|
|scalable한 application 개발하기에 탁월|scalable 애플리케이션 개발에 사용 불가|
|fault tolerant|fault tolerance 보장 불가|

>fault tolerant
>Spark의 가장 기본적인 데이터 추상화 단위인 RDD의 분산 복제 저장, 변환 작업 추적 Lineage Graph, checkpointing을 통해 데이터 손실 시 복구를 보장하는 개념

### 👩‍👧‍👦 4. Spark Component

spark application은 driver process 와 다수의 executor processes로 구성됩니다.

🐔 driver process
- 클러스터 노드 중 **하나**에서 실행됨
- **main() 함수** 실행
- spark application 수명 주기 동안 관련 모든 정보 유지 밀 관리
- 사용자 프로그램/입력 응답, executor process의 job 관련 분석, 배포, 스케줄링

🐤 executor process
- 클러스터의 **여러 머신**에서 실행되는 작업 실행 프로세스
- 드라이버 프로세스가 할당한 **작업(코드) 실행**, 진행 상황 드라이버 노드에 보고
- 각 익스큐터는 클러스터 노드에서 실행되는 JVM(Java Virtual Machine) 프로세스로, 메모리와 CPU를 할당받아 작업을 병렬로 처리

두 프로세스는 사용 가능한 자원을 파악하기 위해 Cluster manager를 사용합니다. Cluster manager는 물리적 머신을 관리하고 스파크 애플리케이션에 자원을 할당하는 소프트웨어로, standalone, Hadoop Yarn, Mesos, Kubernetes 가 있습니다. 

### 🏋️‍♀️ 5. Spark Execution Modes

#### 1. shell mode - interactive mode. direct manipulation in a shell (`spark-shell`, `pyspark`)
#### 2. local mode 
- non-distributed single JVM deployment mode. pseudo cluster
- parallelism defined by paramenter in a spark master url is the numner of threads
- simplt setup, get immediate feedback, cluster management is not required
- development, testing, debugging purpose

![240630_fig1-2](https://github.com/brickstudy/Docs/assets/52881652/3939603f-ca18-4813-b1f6-5d19f91a176c)

#### 3. cluster mode 
- cluster mode is for connecting into private network with several machines. 
- deploying on a cluster, running via cluster manager by submitting application
- `./bin/spark-submit --class <main-class> --master <master-url> --deploy-mode <deploy-mode> --conf <key>=<value> ... other options <application-jar> [application-arguments]`

![240630_fig3](https://github.com/brickstudy/Docs/assets/52881652/d3f075a0-e17f-4408-ad3b-c18488233969)

- there's two kinds of deploy mode : client mode, cluster mode
- 3-1. client mode (default)
    - driver **runs in a same process** as client that submits the app
    - driver program runs on a client machine **outside** the cluster (local machine from which the user submits the job)
    - users can directly check the driver logs on their local machine
    - development, debugging purpose
- 3-2. cluster mode
    - driver launched from a worker process
    - client process exits immediately after application submission
    - driver program이 cluster 내 nodes 중 하나에서 실행됨 즉, cluster manager가 driver program 실행할 노드를 결정함.
    - driver program runs on one of the nodes within the cluster, meaning that cluster manager decides on which node the driver program will execute
    - reduce network latency and allows for better resource management
    - driver logs must be checked on a node within the cluster
    - for production, huge data processing

![240630_fig4](https://github.com/brickstudy/Docs/assets/52881652/71aa73aa-57ed-4afc-be6b-80041f0f32c4)

## Main Content


### main development environment

해당 로컬 작업 환경은 docker를 사용하여 개인 랩탑에서 팀원 모두 공통된 개발 환경을 쉽게 셋팅할 수 있게 개발되었습니다.
공통된 작업 환경은 팀 내에서 일관되고 표준화된 환경을 사용하기 때문에 필요한 도구 및 자원을 명시적이고 공통되게 정의할 수 있고, 문제 해결과 커뮤니케이션 코스트를 줄일 수 있다는 장점이 있습니다.
databricks에서 제공하는 환경의 경우 비용이 상당하게 발생하고 community edition의 경우 제공한 클러스터를 2시간 주기로 회수하기 때문에, 테스트 목적으로 개발하는 상황에 적합한 환경을 제공하는 것을 목표로 했습니다.

공통 로컬 작업 환경은 크게 0. Python 1. 대화형으로 쉽게 작업할 수 있는 Jupyter Notebook 2. 메인 데이터 처리 엔진 Spark 3. AWS 클라우드 서비스 연결 을 dockerize 하여 배포해두었습니다.
해당 베이스 이미지는 ubuntu에 python, conda, jupyter, scipy package, spark 순서 레이어로 구성됩니다. (도커 이미지 계층구조 figure6 참고)

![240630_fig6](https://github.com/brickstudy/Docs/assets/52881652/f10b4712-e78f-477c-aee1-644a2b98ac87)



### 개발 환경 분리

A. (Local mode) 메인 환경! Notebook에서 스파크를 이용한 데이터 처리 코드를 interactive하게 실행하고 실습하기 위한 환경<br>
B. (Cluster mode - standalone) spark-submit으로 스크립트 제출하여 spark job을 실행시키는 환경

![240630_fig5](https://github.com/brickstudy/Docs/assets/52881652/5b96eb36-79fc-4e6b-8693-35263f4f0b12)

환경 분리 context

초기 컴포즈 파일 의 경우 master, worker spark-cluster와 주피터 컨테이너를 띄워서 주피터 상에서 spark-cluster 내의 스파크 엔진을 standalone cluster mode로 사용하게끔 설정이 되어있습니다. 

하지만 jupyterlab 컨테이너 자체에도 spark 가 설치되어있기 때문에, 노트북으로 실습하는 용도로 환경을 구성하는데에 spark-master, spark-worker 인스턴스를 항상 함께 빌드하는게 불필요하다고 판단했습니다. 따라서 단순 pyspark 코드가 실행되는지 실습할때에는 jupyterlab 컨테이너만 띄워 local mode(no cluster manager) 로 학습 및 개발이 진행되고, 클러스터의 경우 별도로(docker-compose_prod_test.yml) 띄울 수 있게끔 분리해두어 실제 databricks 클러스터 상에 제출하기 전 local cluster를 띄워 작업 제출을 테스트할 수 있는 환경으로 분리했습니다. 


A. jupyter( + spark)
- 단순 간단하게 스파크 코드 실행해볼 수 있는 환경
- Dockerfile, docker-compose.yml
- setup command : `docker-compose up -d`

B. spark master, spark worker, spark history server
- cluster에 잡을 제출하고 DAG, task schedule 등 확인할 수 있는 환경
- Dockerfile_prod_test, docker-compose_prod_test.yml
- setup command : `docker-compose -f docker-compose_prod_test.yml up`

<br>

### AWS configuration

aws cli를 컨테이너 안에 설치하고, spark 와 aws s3를 연결하여 데이터를 읽어오는걸 정상적으로 테스트했습니다.

aws cli를 사용하여 aws 클라우드 서비스를 사용하기 위해선 각 profile마다  AWS Access Key ID, AWS Secret Access Key, Default region name, Default output format (optional)를 명시해주어야하기 때문에, 컨테이너 베이스 os unix계열 기본 cli가 refer하고있는 경로 ~/.aws 하에 config, credential 파일로 키 값을 넣어주었습니다. (현재 개인 계정으로 테스트) aws cli는 pip 통해서 설치됩니다. (컨테이너 내부 쉘에서 `aws --version` `aws s3 ls` 등의 커맨드 정상 작동 확인 완료)

spark session에 aws s3 연결 관련 기본 구성의 경우 conf/spark-defaults.conf 파일로 관리 가능하게끔 설정했습니다. 다만 aws connector로 사용되는 hadoop-aws Jar, aws-java-sdk-bundle Jar 파일의 경우 스파크 세션 시마다 maven 통해서 받아오게 설정 시 시간이 오래걸려 다운받아서 이미지 자체 jar 경로에 위치시켰습니다.

해당 구성들은 도커 이미지로 빌드하여 dockerhub에 업로드해두었기 때문에, 개인 작업 환경에서는 docker-compose up 커맨드를 통해서 공통된 개발 환경을 셋업할 수 있습니다.


<br>

### guide 

```bash
git clone https://github.com/brickstudy/infra-docs.git

cd spark && docker-compose up -d
```

웹 브라우저 통해 localhost:8888 접근 - jupyter 환경

```python
from pyspark.sql import SparkSession
import os

# spark session 생성 예제
spark = SparkSession.builder\
.appName("spark-local-environment-test")\
.config('spark.jars', '$SPARK_HOME/jars/hadoop-aws-3.2.0.jar')\
.config('spark.jars', '$SPARK_HOME/jars/aws-java-sdk-bundle-1.11.375.jar')\
.getOrCreate()

spark

# aws 연결 예시
s3_uri = "s3a://<uri-to-s3-bucket>"
df = spark.read.format('json').load(os.path.join(s3_uri, 'data/*.json.gz'))
df.show() 
df.printSchema() # basic print command of spark dataframe
display(df)      # enhance readability

```

<br>

### Etcs, Further

이미지 최적화
- 현재 jupyter에서 제공하는 이미지를 사용하기 때문에, customizing에 한계가 있고, 불필요한 패키지가 이미지에 함꼐 빌드되어있거나 필요한 패키지가 존재하지 않을 수 있습니다. 또한 ubuntu base이기 때문에, 경량화 리눅스를 사용하지 않아 이미지 크기가 8-9G로 다소 큽니다.
- base distributed linux os(debian, ubuntu, centos, ..) + python + team configuration 을 가장 상위 베이스 이미지로 설정하고, 필요한 환경 스펙을 논의하여 customed image를 빌드한다면 경량화된 이미지로 관리할 수 있습니다.
- 해당 베이스 이미지에 jvm 및 spark 를 구축하면 보다 깔끔하게 이미지를 관리할 수 있을 것 같다는 생각이 듭니다. 하지만 이 경우 os가 달라짐에 따라 의존성/호환성에 문제가 발생할 수 있고, 테스트가 필요합니다.
<img width="523" alt="240630_fig7" src="https://github.com/brickstudy/Docs/assets/52881652/ecd30e29-d4c6-4552-96ae-b98f420e1669">
