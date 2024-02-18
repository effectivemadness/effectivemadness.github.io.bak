---
layout: post
title:  " Privatelink 로 SaaS(Datadog) 트래픽 private하게 보내기"
date:   2024-02-18 20:48:34+0900
---
회사에서 Observability 가이드/거버넌스 업무를 하고 있는데, 그 중 가장 많이 쓰이는 서비스가 Datadog입니다. 처음 도입 후 이제 꽤 많은 조직에서 사용중인데, 사용중인 조직 중 한 곳에서 우리 서비스 인프라에서 Datadog으로 전송하는 비용을 줄일 수 없을까? 라는 고민을 하고 있어 저희도 여러 고민을 했었습니다. 

그 중 한 방안이 PrivateLink사용 방식입니다. 외부 Outbound 비용으로 과금이 되는 부분을 줄이고 Inter-Region트래픽이나 VPC Peering 비용만 발생하게 된다면 비용 절감을 할 수 있을 것 같아 검토했었고, 실제로 감소할거라 예상했지만 당시 검토 후 이런 저런 이유때문에 적용은 되지 않아 아쉬웠습니다. 하지만 올해 AWS 자격증 중 Advanced Network Specialty를 따기 위해 관련 개념을 정리할 겸 기록을 남겨 봅니다.

## Terms

구성 전, 각 개념들을 정리해보겠습니다.

### PrivateLink

> AWS PrivateLink은(는) VPC를 서비스에 비공개로 연결하여 서비스를 VPC에 있는 것처럼 이용할 수 있는 가용성과 확장성이 뛰어난 기술입니다. - [AWS VPC PrivateLink 문서](https://docs.aws.amazon.com/ko_kr/vpc/latest/privatelink/what-is-privatelink.html)

![privatelink-concepts](/assets/img/A6A1722A-2979-43CA-8C5F-AF83F1093E31.png)

서비스 구조를 보면, 서비스 공급자(흔히 SaaS 서비스 업체)는 어떤 서비스를 배포 후, 그 서비스 앞단 로드밸런서를 골라 엔드포인트 서비스를 생성한 뒤 서비스 사용자(우리)에게 알려주게 됩니다. 서비스 소비자는 서비스 이름을 지정해 VPC엔드포인트를 생성합니다.

다양한 SaaS 서비스들이 PrivateLink 기반의 제품 배포를 지원하고 있는데요, FWaaS(FireWall as a Service)제품군이 대표적입니다. AWS에서도 PrivateLink 기반의 SaaS 제품들을 한 눈에 확인할 수 있도록 AWS MarketPlace에서 관련 기술을 사용하고 있는 서비스를 볼 수 있게 하고 있습니다. 사용자-소비자 모두 AWS 서비스를 사용하는 고객이 되니, AWS입장에서는 더 좋을 나위가 없겠네요.

### Route53 Private Hosted Zone

Route53 Private Hosted Zone(이한 PHZ)는, 특정 VPC내 자원들이 우리가 지정한 DNS 응답을 받도록 설정 가능한 Zone입니다. R53 PHZ를 생성하고, 어떤 VPC를 그 PHZ에 연결한 뒤 우리가 받아야 하는 DNS레코드를 추가합니다. 그 뒤, VPC 안에서 DNS 질의를 날려 보면(EC2에서 nslookup을 한다거나) 우리가 설정한 레코드 대로 응답이 오는 것을 확인할 수 있습니다.

공식 문서를 보면, example.com PHZ를 생성하고 그 아래 db.example.com 레코드를 설정하는것을 예시로 들고 있습니다. 그렇게 설정된 VPC안 EC2에서는 db.example.com 으로 질의를 할 경우, 우리가 설정한 레코드가 응답으로 와 우리가 지정한 DB호스트에 연결 하는 사용례를 들고 있는데요. 비슷한 사용례를 이번 PrivateLink 구성에서도 사용할 예정입니다.

### VPC 간 통신 방법

클라우드 기반 서비스를 구성하다 보면, VPC끼리 통신을 해야 하는 일이 있는데요. VPC간 통신을 하기 위해서 크게 3가지 방법을 검토하게 됩니다. VPC Peering, Transit VPC, Transit Gateway가 그 방법인데 이번 PrivateLink 구성에서는 단순한 구성을 위해 VPC Peering을 사용했습니다.

#### VPC Peering

VPC Peering은, VPC와 VPC를 직접 1:1로 연결하여 통신하도록 합니다. 이 경우 VPC안에서 겹치는 CIDR 대역이 있어서는 안되고, 2개 이상의 VPC를 연결해 통신할 경우 모든 VPC가 성형으로 연결되어야 합니다. VPC Peering의 장점은 레이턴시가 가장 낮다는 점, 대역폭 제한이 없다는 점, 통신된 트래픽만큼 비용이 발생한다는 점이 있습니다. 다만, VPC 수가 증가하면 복잡도가 증가하고, VPC당 125Peer가 한계라는 단점이 있습니다.

![vpc-peering](/assets/img/46BD9F03-83D4-486C-9AE5-9170EC05092B.png)

#### Transit VPC

Transit VPC는 한 특정 VPC를 Hub로 하여, 다른 VPC를 그 VPC에 VPN으로 연결해 각 VPC끼리의 통신을 Hub VPC 내 EC2기반 가상 라우터에서 하는 구성을 띕니다. EC2에 가상 라우터를 만들어야 해 관리 포인트가 증가하고, 터널 기술 오버헤드가 발생합니다. 통신 트래픽 비용에 EC2 비용, VPN 비용이 발생합니다.

![transit-vpc](/assets/img/130392F0-10F4-4594-A61F-CBA548EF703D.png)

#### Transit Gateway

Transit Gateway는 생성 후, VPC를 Attach하게 되면 TGW에 연결되는 VPC끼리 통신을 할 수 있는 AWS 자원입니다. AWS 관리형 서비스라 관리 포인트가 적습니다. AZ당 100Gbps 대역폭으로 통신할 수 있고, TGW 연결시 시간당 Attatchment 비용이 발생합니다.

![transit-gateway](/assets/img/334F3ED5-8CB3-4738-AFA6-57E572801F1A.png)

## 구성

구성은 Datadog 공식 문서를 사용해 진행했습니다. 공식 문서가 친절하게 나와있어 Step-by-step으로 어떻게 했는지 남기지는 않지만, 크게 다섯 단계로 구성했습니다.

1. PrivateLink 자원을 생성할 Region에 VPC(VPC P)와 서브넷 생성
2. VPC P에 생성할 PrivateLink에 적용될 Security Group 생성
3. VPC P내 서브넷에 PrivateLink 엔드포인트 생성 + SecurityGroup 적용
4. 연결할 서비스가 있는 VPC(VPC S)와 VPC P간 VPC Peering 생성(+ 서로 통신하기 위한 Route Table 추가)
5. Datadog 도메인 쿼리시, PrivateLink 엔드포인트로 응답해주는 Route53 Private Hosted Zone 생성

Datadog의 PrivateLink 엔드포인트가 us-east-1 리전에서만 생성이 가능해, us-east-1리전의 VPC를 만들고 그 VPC까지 Peering으로 연결이 필요했습니다.

구성 후 전체적인 그림은 아래와 같았습니다. 

![privatelink-saas](/assets/img/98CD1220-A3A5-4FFD-8FDA-5DDA3B2CFD93.png)

구성 후 원래 목적인 비용을 계산해본 결과, SaaS측으로 전송되는 비용이 아주 많은 경우 트래픽 비용을 줄일 수 있었습니다. 다만, VPC 생성과 Peering 등 각 서비스에서 네트워크 자원 관련된 F/up이 필요한 점이나 구성 복잡도 관련된 이슈로 바로 적용하지는 못했습니다 :(

## 결론

비용때문에 시작한 검토이긴 하지만, 본 태스크를 진행하며 AWS 네트워크 자원에 대해 조금 눈을 뜨게 된것 같습니다. AWS SAA 자격증 공부할때도, 네트워크는 그냥 연결하면 됩니다~ 이런 스탠스로 주로 각 AWS서비스에 대한 설명이나 소개가 주 내용이어서 이번에 다양한 네트워크 개념을 알 수 있어 좋았습니다. 아쉽게도 이번에는 원하던 곳에 원하던 목적으로 적용하지 못했지만, 사내 공유 후 다른 곳에서 검토해보겠다고 여쭤보셔서 뿌듯하기도 했네요.

VPC Peering같은 경우, 개념은 이미 알고 있었지만 실제로 Peering을 맺으면서 직접 Routing table을 넣어 준다거나 연결을 하기 위해 리전을 오가며 요청-승낙 프로세스를 해보니 AWS등 클라우드가 나오기 전에는 이 과정을 전부 물리적으로 했다는 점이 새삼 놀랍게 느껴졌습니다.

더해, R53 PHZ 구성을 해 보면서도 적용 전후 바로 EC2 인스턴스에서 nslookup 결과가 바뀌는것을 보며 당연히 되는건데 신기했습니다. 개인 프로젝트를 할 때도 /etc/hosts 를 건드린다던가, Domain 구매 후 제 IP를 연결할 때 Domain 판매업체 nameserver에 레코드를 넣고 결과를 확인해본적이 있지만 이번에 R53이라는 한꺼풀 추상화된 자원을 써보며 새롭게 다가왔던 것 같습니다.

PrivateLink같은 경우, 사내에서 자주 사용되는 서비스를 분리해 구성하고 PrivateLink로 노출시켜 다른 AWS 계정이나 VPC에서 사용하는 방법 등으로 활용할 수 있을 것 같습니다. 같은 리전에서 사용하게 되면 inter-region 비용이 발생하지 않아 비용도 큰 부담이 없을 것 같구요. 