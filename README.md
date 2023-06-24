# QuitBoard Service

 QuitBoard는 급증하는 전동스쿠터 사고율의 인식을 전환하고자 만든 비영리 목적의 프로젝트입니다.  
cctv영상을 통해 전동스쿠터와 관련된 여러 위반 사항을 감지하여 초상권을 준수하는 선에서 위반 정보를 제공합니다.\

### 동기

  제도와 현실 사이의 타협점이 모호한(헬멧을 써야하지만 헬멧이 없는 킥보드가 많거나 쓰지 않는 상황 등) 점 때문에 직접적인 신고나 단속이 없습니다. 이런 점을 생각해 직접적인 규제를 하지않고 웹서비스를 통해 위반 사항을 카테고리 별로 제공함으로써, 통계 표시를 통한 간접적인 규제 효과를 기대하기 위한 프로젝트입니다.

###  웹페이지가 데이터를 제공하는 방식

1. CCTV에서 기록된 영상을 딥러닝 모델에 넣는다.
2. 딥러닝 모델에서 영상을 돌려 얻은 분석 결과를 zip파일로 만든다.
3. 만들어진 zip파일을 관리자 페이지를 통해 업로드한다.
4. 업로드된 zip파일을 분석하여 jpg, gif 등 이미지를 저장하고 분석 결과를 데이터베이스에 저장한다.
5. 웹페이지에서  여러 카테고리(장소, 시간, 위반사항)를 선택하여 위반 데이터를 카드 형식으로 제공한다.



# 기술 스택

**Backend** 

- Python
- Django, DRF, Celery
  - 주력 언어인 python을 바탕으로 Django를 선택했고 RESTful API를 위한 편의 기능이 제공되는 DRF를 사용
  - 대용량 데이터를 파싱하기 위한 비동기처리 시스템인 celery를 사용
- Visual Studio Code
- ubuntu 22.04

**DevOps** 

- AWS EC2, S3, RDS, CloudWatch, Route53
  - AWS 환경에서 서버를 배포하고 다양한 서비스를 활용
- gunicorn, Nginx
  - 별다른 에러가 없이 django와 호환이 잘되는 미들웨어인 gunicorn을 사용
  - gunicorn과 잘  연동되고 소규모 서버로서 적절한 역할을 가질 수 있는 Nginx를 사용
- Redis
  - Nginx에서 단일 프로세스 및 스레드 사용으로 CPU 부담을 줄인 부분을 캐시 메모리로 활용할 수 있었기에 인메모리 DB인 redis를 사용
- MySQL
  - DB모델 자체가 단순하고 데이터의 누락 가능성이 적은 정형데이터이기 때문에 관계형 데이터베이스를 선택
  - 기존 데이터의 수정이 적고 



# ERD

위반 데이터 정보를 공공으로 제공하는 것을 목적으로 했기에 단순한 구조의 DB모델을 가지고 있습니다.

### Tabel

- violation : 위반 항목을 저장합니다. 규정이 변화하더라도 대응할 수 있도록 수정가능하게 관리하도록 합니다.
- cctv : cctv정보에 대해 저장합니다. 감시 지역, 상세 주소를 통해 관리 지역을 잘 파악할 수 있도록 하고 cctv모델명을 추가하여 cctv자체 정보를 포함하도록 했습니다.
- violation_info : 위반 데이터입니다. 위반 사항, 위반이미지, 위반시각 등의 위반에 대한 필요한 정보를 포함합니다.
- violation_file : cctv 영상 데이터입니다. 관리자가 영상 파일 자체를 업로드하고 관리할 수 있도록 별도의 테이블로 두었습니다.
- user : 공공데이터를 제공하는 것이 목적이므로 사용자는 없지만 관리자를 위해 간단하게 구현했습니다.



# API 명세서

| Method | URL                         | Description                                          |
| ------ | --------------------------- | ---------------------------------------------------- |
| GET    | /                           | 홈페이지                                             |
| GET    | /features                   | 홈페이지 설명                                        |
| GET    | /contact                    | 개발팀 소개페이지                                    |
| GET    | /statistic                  | 위반사항 전체 현황 요약, 시각화                      |
| GET    | /violations                 | 위반 사항을 보기위한 구체적인 항목(kind) 선택 페이지 |
| GET    | /violations/{kind}          | 선택한 항목에서 추가 항목(detail) 제공               |
| GET    | /violations/{kind}/{detail} | 선택 항목에 맞는 위반 데이터를 카드 형식으로 제공    |
| *      | {aws_ec2_domain}/admin      | django관리자 페이지                                  |



# 개선사항

- **AWS Lambda를 통한 zip파일 처리**
   현재 구현된 사항으로는 1일~2일 분량의 CCTV 영상을 가져와서 로컬에서 딥러닝모델을 돌려 zip파일을 가져오는 형식입니다. 이 때문에 대용량 파일을 처리하기 위해 celery를 통해 비동기로 데이터를 처리하면서 CPU 부담이 올라가고 이 때 트래픽이 겹쳐서 올라간다면 서버가 다운되는 현상이 나타납니다.
   이를 위해 딥러닝 출력 결과를 저장하지 않고 곧바로 AWS Lambda 서비스로 전송하여 Lambda에서 데이터를 파싱한다면 zip파일을 저장하고 다시 풀고 할 필요가 없어서 저장소에 여유가 생기고 서버의 CPU 부담도 줄어들 수 있다고 생각합니다. 
- **캐시 메모리 활용 방법** 
   현재는 가장 많이 조회가 가능한 전체 현황 요약 페이지에만 캐시 조회가 구현되어 있습니다. 전체 현황은 1일 단위로 업데이트가 되는 방향이라 초기부터 캐시 조회를 구현했습니다. 카테고리는 나뉘어져 있지만 제공되는 위반데이터는 결국 같습니다.
   그렇다면 위반데이터를 캐시로 두는 것이 좋지만 NoSQL인 redis이기에 Key 형식으로 각 데이터를 저장하기 어렵습니다. 따라서 카테고리별 데이터를 pagenation을 하여 페이지별 데이터 캐시를 두는 것으로 구현하는 것을 개선점으로 둘 수 있습니다.

- 
