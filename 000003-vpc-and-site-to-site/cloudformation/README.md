# 000003 - Start with Amazon VPC and AWS VPC Site-to-Site

## 1. Preparation Steps
1. Create VPC
2. Create Subnet
3. Create Internet Gateway
4. Create Route Table
5. Create Security Group

### 1.1 Create VPC

Trong bài lab này, phần tạo VPC chúng ta có các thông tin như bên dưới 

**TABLE 1** - Thông tin của ASG VPC
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Resource | VPC Only |
| 2 | Name Tag | ASG |
| 3 | IPv4 CIDR | 10.10.0.0/16 |
| 4 | DNS Resolution | Disabled | 
| 5 | DNS Hostnames | Disabled |


Tài nguyên chúng ta cần tạo là VPC. Tài liệu hướng dẫn của CloudFormation dành cho resource này tại [đây](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html) 

Với các thông tin ở trên, chúng ta sẽ có block resource cho VPC cần tạo như sau

```
Resources:
  ASGVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.10.0.0/16
      EnableDnsSupport: false
      EnableDnsHostnames: false
      Tags: 
        - Key: Name
          Value: ASG
```


### 1.2 Create Subnet

Bài lap hướng dẫn chúng ta tạo 2 public subnet và 2 private subnet với các thông tin bên dưới

**TABLE 2** - Thông tin của Public Subnet 1
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Subnet name | Public Subnet 1 |
| 2 | Availability Zone | ap-southeast-1a |
| 3 | IPv4 CIDR block | 10.10.1.0/24 |


**TABLE 3** - Thông tin của Public Subnet 2
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Subnet name | Public Subnet 2 |
| 2 | Availability Zone | ap-southeast-1b |
| 3 | IPv4 CIDR block | 10.10.2.0/24 |

**TABLE 4** - Thông tin của Private Subnet 1
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Subnet name | Private Subnet 2 |
| 2 | Availability Zone | ap-southeast-1b |
| 3 | IPv4 CIDR block | 10.10.3.0/24 |

**TABLE 5** - Thông tin của Private Subnet 2
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Subnet name | Private Subnet 2 |
| 2 | Availability Zone | ap-southeast-1b |
| 3 | IPv4 CIDR block | 10.10.4.0/24 |

Tài liệu hướng dẫn của CloudFormation của Subnet tại [đây](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html)

Và 2 Public Subnet được cấu hình cho phép tự động asign Public IP. Vì vậy Properties `MapPublicIpOnLaunch` cần được đặt thành `true`

Với 4 bảng trên, chúng ta sẽ có 4 block yaml sau:

```
ASGPublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ASGVpc
      AvailabilityZone: ap-southeast-1a
      CidrBlock: 10.10.1.0/24
      MapPublicIpOnLaunch: true
      Tags:
      - Key: Name
        Value: Public Subnet 1

  ASGPublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ASGVpc
      AvailabilityZone: ap-southeast-1b
      CidrBlock: 10.10.2.0/24
      MapPublicIpOnLaunch: true
      Tags:
      - Key: Name
        Value: Public Subnet 2

  ASGPrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ASGVpc
      AvailabilityZone: ap-southeast-1a
      CidrBlock: 10.10.3.0/24
      MapPublicIpOnLaunch: false
      Tags:
      - Key: Name
        Value: Private Subnet 1

  ASGPrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref ASGVpc
      AvailabilityZone: ap-southeast-1b
      CidrBlock: 10.10.4.0/24
      MapPublicIpOnLaunch: false
      Tags:
      - Key: Name
        Value: Private Subnet 2
```

### 1.3 Create Internet Gateway

Thông tin của Internet Gateway trong bài lab như sau:

**TABLE 6** - Thông tin của Internet Gateway
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Name tag | Internet Gateway |

Từ docs và các thông tin trên ta có block resource để tạo Internet Gateway như sau

```
InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
      - Key: Name
        Value: Internet Gateway
```

Sau đó chúng ta cần attach Internet Gateway này vào VPC đã tạo

```
ASGVpcGatwayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref ASGVpc
      InternetGatewayId: !Ref InternetGateway
```

> Lưu ý: !Ref ở đây là cú pháp để references tới 1 resource trong template của CloudFormation. Trước đó chúng ta đã tạo ASGVpc và InternetGateway. Và theo document của 2 tài nguyên này thì Ref return value là ID của resource.

### 1.4 Create Route Table

**TABLE 7** - Thông tin của Route Table
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Name | Route table-Public |
| 2 | VPC | VPC ID đã tạo |

```
ASGVpcRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref ASGVpc
      Tags:
      - Key: Name
        Value: Route table-Public
```

Tiếp theo là chúng ta tạo một `route` trong route table vừa tạo ở trên. để các tài nguyên trong VPC có thể đi ra được internet thông qua Internet Gateway.

```
ASGVpcRouteTableRoute:
    Type: AWS::EC2::Route
    Properties:
        RouteTableId: !Ref ASGVpcRouteTable
        DestinationCidrBlock: 0.0.0.0/0
        GatewayId: !Ref InternetGateway
```

Chỉnh sửa subnet associations của Internet Gateway

```
ASGVpcRouteTableSubnetAssociation1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
        RouteTableId: !Ref ASGVpcRouteTable
        SubnetId: !Ref ASGPublicSubnet1
  
  ASGVpcRouteTableSubnetAssociation2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
        RouteTableId: !Ref ASGVpcRouteTable
        SubnetId: !Ref ASGPublicSubnet2
```

### 1.5 Create Security Group

Chúng ta tiến hành tạo Security Group sử dụng cho các instance trong Public Subnet

**TABLE 8** - Thông tin của Security Group trong Public Subnet
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Security Group name | Public subnet - SG  |
| 2 | Description | Allow SSH and Ping for servers in the public subnet. |
| 3 | VPC | ASG VPC chúng ta đã tạo |
| 4 | Inbound rule | Cho phép ssh và ping từ bất kì địa chỉ ip nào |

**TABLE 9** - Thông tin của Security Group trong Private Subnet
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Security Group name | Private subnet - SG  |
| 2 | Description | Allow SSH and Ping for servers in the private subnet. |
| 3 | VPC | ASG VPC chúng ta đã tạo |
| 4 | Inbound rule | Chỉ cho phép ssh từ các instance trong Public Subnet và cho phép ping từ bất kì địa chỉ ip nào |

```
ASGVpcPublicSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
        GroupDescription: Allow SSH and Ping for servers in the public subnet.
        VpcId: !Ref ASGVpc
        GroupName: Public subnet - SG
        SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 22
              ToPort: 22
              CidrIp: 0.0.0.0/0
            - IpProtocol: icmp
              FromPort: -1
              ToPort: -1
              CidrIp: 0.0.0.0/0
        SecurityGroupEgress:
            - IpProtocol: "-1"
              CidrIp: 0.0.0.0/0
  
ASGVpcPrivateSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
        GroupDescription: Allow SSH and Ping for servers in the private subnet.
        VpcId: !Ref ASGVpc
        GroupName: Private subnet - SG
        SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 22
              ToPort: 22
              SourceSecurityGroupId: !Ref ASGVpcPublicSecurityGroup
            - IpProtocol: icmp
              FromPort: -1
              ToPort: -1
              CidrIp: 0.0.0.0/0
        SecurityGroupEgress:
            - IpProtocol: "-1"
              CidrIp: 0.0.0.0/0
```

## 2. Create EC2 Server

### 2.1 Create EC2 Server

**TABLE 10** - Thông tin của EC2 Instance trong Public Subnet
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Name and Tags | EC2 Public |
| 2 | AMI | AMI Amazon Linux 2 có ID là ami-0adcfe5c27f7c9acf |
| 3 | Instance Type | t2.micro |
| 4 | VPC | VPC ASG đã tạo ở trên |
| 5 | Subnet | Public Subnet 1 |
| 5 | Auto-assign Public IP | Enable |
| 5 | Security Group | Public subnet - SG |


**TABLE 11** - Thông tin của EC2 Instance trong Private Subnet
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Name and Tags | EC2 Private |
| 2 | AMI | AMI Amazon Linux 2 có ID là ami-0adcfe5c27f7c9acf |
| 3 | Instance Type | t2.micro |
| 4 | VPC | VPC ASG đã tạo ở trên |
| 5 | Subnet | Private Subnet 2 |
| 5 | Auto-assign Public IP | Disable |
| 5 | Security Group | Private subnet - SG |

Tuy nhiên để có thể ssh và test connection trong nội dung tiếp theo của bài lab, chúng ta cần tạo key pair để cho thể ssh đến ec2 instances này. Các bước tạo keypair đã có một phần trong bài lab nên mình sẽ không thảo luận nữa. Chỉ cần tạo với name aws-keypair và download keypair về máy local

Sau khi có ID của Keypair thì chúng ta có thể viết template cloudformation để tạo 2 EC2 instance trên như sau:

```
EC2Public:
    Type: AWS::EC2::Instance
    Properties:
        SubnetId: !Ref ASGPublicSubnet1
        ImageId: ami-0adcfe5c27f7c9acf
        InstanceType: t2.micro
        SecurityGroupIds:
            - !Ref ASGVpcPublicSecurityGroup
        KeyName: aws-keypair
        Tags: 
        - Key: Name
            Value: EC2 Public

EC2Private:
    Type: AWS::EC2::Instance
    Properties:
        SubnetId: !Ref ASGPrivateSubnet2
        ImageId: ami-0adcfe5c27f7c9acf
        InstanceType: t2.micro
        SecurityGroupIds:
            - !Ref ASGVpcPrivateSecurityGroup
        KeyName: aws-keypair
        Tags: 
        - Key: Name
            Value: EC2 Private
```

### 2.2 Create NAT Gateway

Trong bài lab, chúng ta cần tạo và gán Elastic IP cho NAT Gateway.

```
EIP:
    Type: AWS::EC2::EIP
    Properties:
        Domain: vpc

NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
        SubnetId: !Ref ASGPublicSubnet2
        ConnectivityType: public
        AllocationId: !GetAtt EIP.AllocationId
        Tags:
        - Key: Name
          Value: NAT Gateway
```

Tiếp theo chúng ta cần cấu hình Route table Private và assign nó cho Private Subnet

**TABLE 12** - Thông tin của Route Table Private Subnet
| # | Tên | Giá trị |
| :-- | :--- | :----|
| 1 | Name | Route table-Private |
| 2 | VPC | VPC ID đã tạo |

```
ASGVpcPrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
        VpcId: !Ref ASGVpc
        Tags:
        - Key: Name
          Value: Route table-Private

ASGVpcPrivateRouteTableSubnetAssociation1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
        RouteTableId: !Ref ASGVpcPrivateRouteTable
        SubnetId: !Ref ASGPrivateSubnet1
  
ASGVpcPrivateRouteTableSubnetAssociation2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
        RouteTableId: !Ref ASGVpcPrivateRouteTable
        SubnetId: !Ref ASGPrivateSubnet2
```

Tạo route entry để route các traffic muốn đi ra internet (0.0.0.0/0) thì đi qua NAT Gateway

```
ASGVpcPrivateRouteTableRoute:
    Type: AWS::EC2::Route
    Properties:
        RouteTableId: !Ref ASGVpcPrivateRouteTable
        DestinationCidrBlock: 0.0.0.0/0
        NatGatewayId: !Ref NATGateway
```

## 3. Setting Up Site-to-Site VPN Connection in AWS

... upcoming








