---
layout: post
title:  "AWS NAT Gateway와 Cloudformation Template"
tags: [aws]
---
요즘 작업하는 것들을 대부분 Serverless 프레임워크를 사용해서 AWS Lambda로 백엔드를 구성하고 있다. 환경을 구성하는 데 드는 반복적이고 소모적인 작업들을 제거한다는 측면에서 좋다. 또한 요청이 없을 때도 서버가 돌지 않는다는 점에서 미적으로 더 간결하게 느껴지는 부분도 있다. 얼마 전 학교 수업에서 교수님이 프로그래머는 미적인 감각도 중요하다고 하셨는데, 코드의 아름다움과 서비스의 논리적 완결성을 중요시하는 나에게 힘이 되는 이야기였다.

처음 프로그래밍을 시작할 때 사소한 걸로 골머리를 썩고 시간을 많이 썼었다. 요즘은 웬만한 문제는 금방 해결할 수 있고, 적어도 언제까지는 해결할 수 있겠다는 감이 잡히는 편이다. 그러나 서버 관련한 이슈는 항상 시간이 오래 걸린다. AWS가 너무 복잡한 탓이라고 생각한다. 문서도 그리 친절하지 않다. 워낙 방대한 서비스를 하다보니 문서 하나하나를 그렇게 자세히 만들 수도 없고, 딱히 대체재도 없는 탓에 그런대로 다들 시간을 들여 쓸 수 밖에 없으니 보완할 필요성이 없는지도 모르겠다.

저번에 Lambda 함수가 RDS에 접근하지 못하는 점을 개선하기 위해 VPC 설정을 했었는데, VPC에 속하면서 AWS의 서비스 간 액세스가 가능해진 동시에 인터넷 액세스가 불가능해졌다. 그런데 이번에 외부에 API 콜을 해야하는 함수를 구현해야 했고, 따라서 AWS의 서비스 간 액세스와 동시에 인터넷 액세스도 되게 하는 방법을 찾아야했다.



# AWS 서비스 간 연결과 인터넷 연결을 동시에 하기 위해서는 NAT Gateway 설정이 필요하다.

> API 문서를 정독했다고 생각하는데도 어려웠다. 이해가 되지 않는 개념에 대한 텍스트는 쓰인 그대로 읽을 수가 없는 것 같기도 하다.

서비스들을 같은 VPC(Virtual Private Cloud)에 묶어두면 해당 VPC에 속한 서비스들 간에는 액세스가 가능하다. 그러나 이렇게 되면서 인트라넷 망에 묶는 것마냥 인터넷 액세스가 안된다. 이 때 인터넷 연결이 필요하다면 NAT Gateway라는 걸 설정해야 한다. NAT란 Network Address Translation의 약자로 [위키백과](https://ko.wikipedia.org/wiki/%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC_%EC%A3%BC%EC%86%8C_%EB%B3%80%ED%99%98)에서는  대개 사설 네트워크에 속한 여러 개의 호스트가 하나의 공인 IP 주소를 사용하여 인터넷에 접속하기 위해 쓰인다고 설명하고 있다. 포트 숫자를 재기록한다니 어쩐다니 하는 말이 있는데 아직 잘 모르겠고 얼른 자야해서 나중에 공부할 예정이다.

결국 [이 gist](https://gist.github.com/reggi/dc5f2620b7b4f515e68e46255ac042a7)를 참고하여 해결했는데, 내 Lambda 함수에는 RDS 연결을 위한 기존 Subnet이 있어서 설명이 좀 헷갈렸다. 새로운 Subnet을 만들어 이것을 Internet Gateway에 연결하고, 이 Subnet과 연결되는 NAT Gateway를 만든다. 그리고 기존 Subnet이 있다면 그 Subnet들을 이 NAT Gateway로 연결되게 만들고, 없다면 새로운 Subnet을 만들어서 이 NAT Gateway에 연결한다. 새로 만들었다면 이 Subnet들을 Lambda 함수에 연결해주는 것도 필요하다. 이렇게 하면 Lambda에 연결된 Subnet들은 저 NAT Gateway로 연결되고, NAT Gateway는 인터넷에 액세스할 수 있기 때문에 NAT Gateway를 거쳐 인터넷에 연결되는 구조로 추측된다.

직관적으로 생각하면 이 구조가 맞는데 뭔가 더 복잡할 거라고 생각하고 다른 짓들을 했었다. Lambda에 연결하는 Subnet을 Internet Gateway와 연결하기도 했었고, Lambda에 Internet Gateway와 연결된 Subnet과 NAT Gateway와 연결된 Subnet을 다 연결하기도 했었다. (이건 그래도 될 것 같지 않나?)

RDS는 NAT Gateway에만 연결하면 로컬에서 접속이 불가능해진다. 지금은 로컬에서 이런저런 작업들을 해야하기 때문에 Internet Gateway와의 연결이 필요하고, 따라서 RDS는 Lambda와 연결하기 위한 Subnet(NAT Gateway와 연결되어 있는), 그리고 인터넷 연결을 위한 Subnet(Internet Gateway와 연결되어 있는)을 모두 연결하면 람다에서의 액세스와 로컬에서의 액세스가 모두 가능해진다.



# CloudFormation Template으로 필요한 서버 설정들을 명시해두기

지금 서버가 원하는 대로 돌아가고 있다는 것만으로는 부족하다. 이렇게 원하는 대로 돌아가는 서버의 구성이 어떻게 되어있는지를 알고 그것을 재현할 수 있어야 한다. 왜냐면, 첫째로 서버를 재설정할 필요가 생길 수 있다. 현재 구성한 스택은 개발 단계의 서버이다. 프로덕션 단계로 넘어가면서 스택을 다시 만들어야하는데 이 때 현재 구성과 동일한 서버를 다시 만들 수 있어야 한다. 둘째로는 설정을 변경할 때를 위해서이다. 어떤 변화를 줄 때 기존의 설정과 의존관계가 있을 수 있는데 이를 올바르게 설정하기 위해서는 기존에 내가 서버에 무슨 짓들을 해 두었는지를 정확하게 알아야 한다.

Serverless 프레임워크의 설정 파일인 `serverless.yml`의  Resource라는 항목에는 AWS CloudFormation Template이 들어간다. AWS에서 요구하는 양식에 맞게 써 넣으면 이대로 CloudFormation이 구성된다. 이 템플릿만 있으면 VPC, RDS 등 AWS가 제공하는 다양한 리소스들로 구성된 스택을 한 번에 만들 수 있다. [이 문서](http://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)에서 가용 리소스들과 템플릿에 들어가야 할 항목들을 확인할 수 있다.



# 불만 (혹은 내가 놓치고 있는 부분)

RDS에 Subnet Group에 NAT Gateway에 연결된 Subnet 들만 포함되어 있으면 외부로부터 RDS로 Inbound가 안된다. Internet Gateway에 연결된 Subnet을 포함시키면 되는데, 이상한 점은 포함시켰을 때는 접속이 안되다가 임의의 VPC를 생성해 이 VPC 안의 임의의 SubnetGroup으로 옮겼다가 다시 이 SubnetGroup을 할당하면 접속이 된다. 변경 사항이 반영되지 않고 있을 가능성이 있다는 어떤 메시지도 나타나지 않는데 매우 불편한 부분이다.

직접 NAT Gateway를 설정하고 위와 같이 InternetGateway를 연결할 때와 동일한 설정을, CloudFormation Template으로 만들어서 디플로이하면 또 액세스가 안된다. 아직까지는 직접 설정한 것과 다른 점을 찾지 못했는데 이것도 매우 이상한 부분이다.
