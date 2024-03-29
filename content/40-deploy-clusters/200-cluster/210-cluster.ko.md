---
title: 기본 클러스터 스택 정의
weight: 210
---


## EKS 클러스터 생성하기

다음과 같이 import된 패키지를 인스턴스화하여 클러스터를 정의합니다.
`const primaryRegion = 'ap-northeast-1';` 선언 아래 다음 코드를 복사하여 붙여주십시오.
```typescript
    const clusterAdmin = new iam.Role(this, 'AdminRole', {
      assumedBy: new iam.AccountRootPrincipal()
      });

    const cluster = new eks.Cluster(this, 'demogo-cluster', {
        clusterName: `demogo`,
        mastersRole: clusterAdmin,
        version: '1.14',
        defaultCapacity: 2
    });

    cluster.addCapacity('spot-group', {
      instanceType: new ec2.InstanceType('m5.xlarge'),
      spotPrice: cdk.Stack.of(this).region==primaryRegion ? '0.248' : '0.192'
    });


```


* `clusterAdmin`은 여러분의 클러스터에 `kubectl` 등의 명령어를 수행할 때 assume 할 IAM Role 입니다.
* `cluster`가 우리가 생성할 EKS 클러스터입니다. [이 가이드](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-eks.Cluster.html)에 따라 여러가지 클러스터 값 설정을 할 수 있는데요, 이 워크샵에서 우리는 다음과 같은 설정을 할 것입니다.
    * `clusterName`: 세 번째 랩에서 사용하기 위해 우리는 이름을 미리 지정했습니다. 이 이름은 한 리전 내에서 고유해야 합니다. 입력하지 않을 경우 CDK가 자동으로 이름을 생성합니다.
    * `mastersRole`: kubectl 조작 권한을 가질 수 있게 해주는 Kubernetes RBAC 그룹 `systems:masters`에 추가될 IAM 주체를 선언합니다. 우리는 위에서 정의한 `clusterAdmin`을 이용해서 해당 클러스터에 접근하기 위해 이 롤을 입력합니다.
    * `defaultCapacity`: 기본으로 몇 개의 워커노드가 생성될 것인지 지정합니다. 
* `cluster.addCapacity`: 기본으로 잡아둔 워커노드에 더해 EC2 Spot 인스턴스를 활용하는 워커노드를 추가로 별도 AutoScalingGroup으로 추가했습니다. (2020년 5월 기준, Managed Nodegroup에서 스팟 인스턴스를 지원하지 않아 ASG로 작업합니다.)

{{% notice info %}} 
참고: EC2 스팟 인스턴스는 더 이상 bidding 이 아니라, [market price 로 구매](https://aws.amazon.com/ko/blogs/compute/new-amazon-ec2-spot-pricing/)를 하게 됩니다.  
그래서 IaC로 스팟 인스턴스를 생성/관리하실 때에는 가격 부분을 온디맨드 인스턴스 가격으로 지정해두시면, 알아서 해당 시점의 스팟 가격으로 구매가 이루어집니다.
{{% /notice %}}



완성된 코드는 다음과 같을 것입니다.

``` typescript
import * as iam from '@aws-cdk/aws-iam';
import * as eks from '@aws-cdk/aws-eks';

export class ClusterStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    const primaryRegion = 'ap-northeast-1';
    const clusterAdmin = new iam.Role(this, 'AdminRole', {
      assumedBy: new iam.AccountRootPrincipal()
      });

    const cluster = new eks.Cluster(this, 'demogo-cluster', {
        clusterName: `demogo`,
        mastersRole: clusterAdmin,
        version: '1.14',
        defaultCapacity: 2
    });

    cluster.addCapacity('spot-group', {
      instanceType: new ec2.InstanceType('m5.xlarge'),
      spotPrice: cdk.Stack.of(this).region==primaryRegion ? '0.248' : '0.192'
    });

    //...
```


## 엔트리포인트에 스택 로드하기
그러면 우리가 완성한 이 스택이 실제로 AWS클라우드에서는 어떻게 보일까요?  
**bin/multi-cluster-ts.ts** 파일을 열어 스택을 로드해보겠습니다.
아래와 같이, 스택 생성에 사용할 AWS 계정과 Region 정보가 정의되어 있습니다.  
이 실습에서는 **도쿄 리전(ap-northeast-1)** 과 **오레곤 리전(us-west-2)** 을 사용해서 자원을 생성할 것입니다.  
필요하다면 리전을 변경하여 주십시오.

```typescript
const account = app.node.tryGetContext('account') || process.env.CDK_INTEG_ACCOUNT || process.env.CDK_DEFAULT_ACCOUNT;
const primaryRegion = {account: account, region: 'ap-northeast-1'};
const secondaryRegion = {account: account, region: 'us-west-2'};
```


아래 코드를 추가하여 스택을 로드한 뒤 저장해주십시오.
```typescript
const primaryCluster = new ClusterStack(app, `ClusterStack-${primaryRegion.region}`, {env: primaryRegion })
```



## cdk bootstrap

이 워크샵은 도쿄 리전(ap-northeast-1)과 오레곤 리전(us-west-2)에 자원을 배포할 것입니다.  
이 두 리전에 대해 `cdk bootstrap` 작업을 수행해주어야, 우리가 워크샵을 통해 작성하는 CDK 코드를 이용해 AWS 자원을 만들 수 있습니다.

cdk bootstrap 은 CDK가 특정 환경(계정, 리전)에 자원 배포를 수행하기 위해 필요한 설정을 하도록 도와주는 AWS CDK cli 입니다. cdk bootstrap 을 수행하면 CDK toolkit을 위한 스택이 AWS 환경에 배포됨을 확인할 수 있습니다. 이를 통해 CloudFormation 템플릿과 기타 asset을 저장하는 S3 bucket이 생성됩니다.

```
ACCOUNT_ID=$(aws sts get-caller-identity|jq -r ".Account")
cdk bootstrap aws://$ACCOUNT_ID/ap-northeast-1
cdk bootstrap aws://$ACCOUNT_ID/us-west-2
```

{{% notice info %}} 
ssh 세션을 맺어 다른 서버에서 작업하시는 경우, 세션이 만료되지 않도록 한 차례 갱신한 뒤 수행하실 것을 권고드립니다. 
{{% /notice %}}




## 생성할 자원 확인하기
이제 다음 명령어를 사용해 어떤 자원들이 생성될 지 확인해보겠습니다.
{{% notice info %}}
`cdk diff`는 실제 자원을 배포하기 전에 기존 대비 어떤 자원들이 새롭게 생기거나 변경, 삭제될 것인지 보여주는 명령어입니다.
{{% /notice %}}
```
cdk diff
```

{{% notice warning %}}
만약 아래와 같이 결과가 출력되지 않는다면, [이 단계](/ko/40-deploy-clusters/100-clone-base-repo/)를 참고하여 `npm run watch`가 백그라운드에서 수행 중인지 확인하십시오.
{{% /notice %}}


아래와 같은 결과가 출력될 것입니다.
```
Stack ClusterStack-ap-southeast-2
IAM Statement Changes
┌───┬────────────────────────┬────────┬────────────────────────┬────────────────────────┬───────────┐
│   │ Resource               │ Effect │ Action                 │ Principal              │ Condition │
├───┼────────────────────────┼────────┼────────────────────────┼────────────────────────┼───────────┤
│ + │ ${AdminRole.Arn}       │ Allow  │ sts:AssumeRole         │ AWS:arn:${AWS::Partiti │           │
│   │                        │        │                        │ on}:iam::<<ACCOUNT_ID>>: │           │
│   │                        │        │                        │ root                   │           │
...
...
...
...
Resources
[+] AWS::IAM::Role AdminRole AdminRole38563C57 
[+] AWS::EC2::VPC demogo-cluster/DefaultVpc demogoclusterDefaultVpc0F0EA8D6 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PublicSubnet1/Subnet demogoclusterDefaultVpcPublicSubnet1Subnet9B5D84CC 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PublicSubnet1/RouteTable demogoclusterDefaultVpcPublicSubnet1RouteTableA9719167 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PublicSubnet1/RouteTableAssociation demogoclusterDefaultVpcPublicSubnet1RouteTableAssociationF6BCC682 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PublicSubnet1/DefaultRoute demogoclusterDefaultVpcPublicSubnet1DefaultRoute0A0FDBF1 
[+] AWS::EC2::EIP demogo-cluster/DefaultVpc/PublicSubnet1/EIP demogoclusterDefaultVpcPublicSubnet1EIP42D57092 
[+] AWS::EC2::NatGateway demogo-cluster/DefaultVpc/PublicSubnet1/NATGateway demogoclusterDefaultVpcPublicSubnet1NATGateway8B5F277B 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PublicSubnet2/Subnet demogoclusterDefaultVpcPublicSubnet2Subnet39C69507 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PublicSubnet2/RouteTable demogoclusterDefaultVpcPublicSubnet2RouteTable20C348DA 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PublicSubnet2/RouteTableAssociation demogoclusterDefaultVpcPublicSubnet2RouteTableAssociation8151DA4B 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PublicSubnet2/DefaultRoute demogoclusterDefaultVpcPublicSubnet2DefaultRoute3A2BEF72 
[+] AWS::EC2::EIP demogo-cluster/DefaultVpc/PublicSubnet2/EIP demogoclusterDefaultVpcPublicSubnet2EIPDD1FE783 
[+] AWS::EC2::NatGateway demogo-cluster/DefaultVpc/PublicSubnet2/NATGateway demogoclusterDefaultVpcPublicSubnet2NATGateway9872618E 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PublicSubnet3/Subnet demogoclusterDefaultVpcPublicSubnet3Subnet7AC2FC28 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PublicSubnet3/RouteTable demogoclusterDefaultVpcPublicSubnet3RouteTableB5078316 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PublicSubnet3/RouteTableAssociation demogoclusterDefaultVpcPublicSubnet3RouteTableAssociationDFEE60CD 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PublicSubnet3/DefaultRoute demogoclusterDefaultVpcPublicSubnet3DefaultRoute2D744EC6 
[+] AWS::EC2::EIP demogo-cluster/DefaultVpc/PublicSubnet3/EIP demogoclusterDefaultVpcPublicSubnet3EIPACAD503B 
[+] AWS::EC2::NatGateway demogo-cluster/DefaultVpc/PublicSubnet3/NATGateway demogoclusterDefaultVpcPublicSubnet3NATGateway8A6E1058 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PrivateSubnet1/Subnet demogoclusterDefaultVpcPrivateSubnet1Subnet9EBC7F0F 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PrivateSubnet1/RouteTable demogoclusterDefaultVpcPrivateSubnet1RouteTableF109BA8D 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PrivateSubnet1/RouteTableAssociation demogoclusterDefaultVpcPrivateSubnet1RouteTableAssociation098CAA3A 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PrivateSubnet1/DefaultRoute demogoclusterDefaultVpcPrivateSubnet1DefaultRoute3000F93D 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PrivateSubnet2/Subnet demogoclusterDefaultVpcPrivateSubnet2Subnet71BE4D47 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PrivateSubnet2/RouteTable demogoclusterDefaultVpcPrivateSubnet2RouteTable0695DE49 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PrivateSubnet2/RouteTableAssociation demogoclusterDefaultVpcPrivateSubnet2RouteTableAssociationDA611F45 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PrivateSubnet2/DefaultRoute demogoclusterDefaultVpcPrivateSubnet2DefaultRouteB3F63E45 
[+] AWS::EC2::Subnet demogo-cluster/DefaultVpc/PrivateSubnet3/Subnet demogoclusterDefaultVpcPrivateSubnet3Subnet7C658410 
[+] AWS::EC2::RouteTable demogo-cluster/DefaultVpc/PrivateSubnet3/RouteTable demogoclusterDefaultVpcPrivateSubnet3RouteTableF5EFFC72 
[+] AWS::EC2::SubnetRouteTableAssociation demogo-cluster/DefaultVpc/PrivateSubnet3/RouteTableAssociation demogoclusterDefaultVpcPrivateSubnet3RouteTableAssociation45D2D250 
[+] AWS::EC2::Route demogo-cluster/DefaultVpc/PrivateSubnet3/DefaultRoute demogoclusterDefaultVpcPrivateSubnet3DefaultRoute029BFE42 
[+] AWS::EC2::InternetGateway demogo-cluster/DefaultVpc/IGW demogoclusterDefaultVpcIGWE387E56C 
[+] AWS::EC2::VPCGatewayAttachment demogo-cluster/DefaultVpc/VPCGW demogoclusterDefaultVpcVPCGW187F7878 
[+] AWS::IAM::Role demogo-cluster/Role demogoclusterRole530AF8F4 
[+] AWS::EC2::SecurityGroup demogo-cluster/ControlPlaneSecurityGroup demogoclusterControlPlaneSecurityGroup2C10657D 
[+] AWS::EC2::SecurityGroupIngress demogo-cluster/ControlPlaneSecurityGroup/from ClusterStackapsoutheast2demogoclusterDefaultCapacityInstanceSecurityGroupE5C0D5D8:443 demogoclusterControlPlaneSecurityGroupfromClusterStackapsoutheast2demogoclusterDefaultCapacityInstanceSecurityGroupE5C0D5D84433834DEB2 
[+] AWS::IAM::Role demogo-cluster/Resource/CreationRole demogoclusterCreationRoleB8F24255 
[+] AWS::IAM::Policy demogo-cluster/Resource/CreationRole/DefaultPolicy demogoclusterCreationRoleDefaultPolicy711D53D3 
[+] Custom::AWSCDK-EKS-Cluster demogo-cluster/Resource/Resource demogoclusterA89898EB 
[+] Custom::AWSCDK-EKS-KubernetesResource demogo-cluster/AwsAuth/manifest/Resource demogoclusterAwsAuthmanifestFF93B93C 
[+] AWS::EC2::SecurityGroup demogo-cluster/DefaultCapacity/InstanceSecurityGroup demogoclusterDefaultCapacityInstanceSecurityGroupC4B9CC76 
[+] AWS::EC2::SecurityGroupIngress demogo-cluster/DefaultCapacity/InstanceSecurityGroup/from ClusterStackapsoutheast2demogoclusterDefaultCapacityInstanceSecurityGroupE5C0D5D8:ALL TRAFFIC demogoclusterDefaultCapacityInstanceSecurityGroupfromClusterStackapsoutheast2demogoclusterDefaultCapacityInstanceSecurityGroupE5C0D5D8ALLTRAFFICAB31DDDD 
[+] AWS::EC2::SecurityGroupIngress demogo-cluster/DefaultCapacity/InstanceSecurityGroup/from ClusterStackapsoutheast2demogoclusterControlPlaneSecurityGroupF5A0BAA2:443 demogoclusterDefaultCapacityInstanceSecurityGroupfromClusterStackapsoutheast2demogoclusterControlPlaneSecurityGroupF5A0BAA24434161DB3B 
[+] AWS::EC2::SecurityGroupIngress demogo-cluster/DefaultCapacity/InstanceSecurityGroup/from ClusterStackapsoutheast2demogoclusterControlPlaneSecurityGroupF5A0BAA2:1025-65535 demogoclusterDefaultCapacityInstanceSecurityGroupfromClusterStackapsoutheast2demogoclusterControlPlaneSecurityGroupF5A0BAA21025655350E76CDB2 
[+] AWS::IAM::Role demogo-cluster/DefaultCapacity/InstanceRole demogoclusterDefaultCapacityInstanceRoleD81B758A 
[+] AWS::IAM::InstanceProfile demogo-cluster/DefaultCapacity/InstanceProfile demogoclusterDefaultCapacityInstanceProfile71466EC2 
[+] AWS::AutoScaling::LaunchConfiguration demogo-cluster/DefaultCapacity/LaunchConfig demogoclusterDefaultCapacityLaunchConfig93D71520 
[+] AWS::AutoScaling::AutoScalingGroup demogo-cluster/DefaultCapacity/ASG demogoclusterDefaultCapacityASGCD1D544F 
[+] AWS::CloudFormation::Stack @aws-cdk--aws-eks.ClusterResourceProvider.NestedStack/@aws-cdk--aws-eks.ClusterResourceProvider.NestedStackResource awscdkawseksClusterResourceProviderNestedStackawscdkawseksClusterResourceProviderNestedStackResource9827C454 
[+] AWS::CloudFormation::Stack @aws-cdk--aws-eks.KubectlProvider.NestedStack/@aws-cdk--aws-eks.KubectlProvider.NestedStackResource awscdkawseksKubectlProviderNestedStackawscdkawseksKubectlProviderNestedStackResourceA7AEBA6B

```

여러분은 지금 코드를 대략 20줄 정도 밖에 입력하지 않았는데, 어떻게 이렇게 많은 자원이 생성된 것일까요?  
그 이유는 CDK가 AWS가 권고하는 **베스트 프랙티스를 기본값**으로 갖고 있기 때문에 그렇습니다.  
CDK를 사용하면 CloudFormation이나 Terraform 을 이용하는 것처럼 자원 하나하나를 정의할 수도 있지만,  
**지금처럼 꼭 필요한 부분만 정의하고 나머지 부분은 AWS에게 맡기실 수도** 있습니다.


## 자원 생성하기
```
cdk deploy
```


이 명령어를 수행하고 나면 아래와 같이 여러분의 승인을 요구할 것입니다.  
IAM 등 보안 관련 자원이 생성될 텐데, 이에 대해 동의하느냐는 말입니다.
콘솔에 출력된 결과물을 살펴보신 뒤 `y`를 입력하십시오.
```bash
...
Do you wish to deploy these changes (y/n)? 
```

약 15분 정도의 시간 뒤에 정상적으로 자원이 생성될 것입니다.  
그동안 아래와 같이 콘솔 결과물을 통해, 생성 중인 자원의 상황을 확인할 수 있습니다.

![](/images/20-single-region/creation-inprg.png)


생성이 모두 완료 되면 아래와 같이 [콘솔](console.aws.amazon.com/cloudformation/)에서 CloudFormation 스택이 생성된 것을 확인할 수 있습니다.

![](/images/70-appendix/stacks.png)


{{% notice info %}} 우리는 분명 한 스택만 만들었는데, 왜 이렇게 많은 스택이 생성됐을까요?! 자세한 내용이 궁금하신 분들은 [여기](/ko/80-appendix/how-cfn-addresource/)를 참조하십시오. {{% /notice %}}



## kubeconfig 업데이트하기
![](/images/20-deploy-clusters/stack-output.png)

자원 생성이 완료되고 나면, 위 스크린샷처럼 콘솔에 CloudFormation Output으로 ConfigCommand가 출력될 것입니다.
아래 출력값의 `=` 뒤의 `aws eks ...` 부분을 복사하여 콘솔에서 실행하십시오.  

```
ClusterStack-ap-northeast-1.demogoclusterConfigCommand6DB6D889 = aws eks update-kubeconfig --name demogo --region ap-northeast-1 --role-arn <<ROLE_ARN>>
```

정상적으로 수행되면 아래와 같은 결과가 출력될 것입니다.

```
Updated context arn:aws:eks:ap-northeast-1:<<ACCOUNT_ID>>:cluster/demogo in /<<YOUR_HOME_DIRECTORY>>/.kube/config
```

이를 통해 우리가 방금 생성한 EKS 클러스터에 `kubectl`을 이용하여 자원을 조회/생성/수정/삭제할 수 있습니다.

