Model: # AWS上でReactアプリケーションのCI/CDパイプラインを構築する手順

GitHubリポジトリのReactアプリケーションをEC2+ALB構成にデプロイするCI/CDパイプラインを構築します。

## 全体の構成

1. GitHubのmainブランチへのpushをトリガーにCodePipelineが起動
2. CodeBuildでReactアプリケーションをビルド
3. CloudFormationでインフラ(VPC、サブネット、ALB、EC2など)をデプロイ
4. CodeDeployでビルドされたアプリケーションをEC2にデプロイ

## 1. サンプルReactアプリケーション

以下のファイルをGitHubリポジトリに作成します。

**src/App.js**
```jsx
import React from 'react';
import './App.css';

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <h1>AWS CI/CD パイプラインデモ</h1>
        <p>これはAWS CodePipeline、CodeBuild、CodeDeployを使用してデプロイされたReactアプリケーションです。</p>
      </header>
    </div>
  );
}

export default App;
```

**src/App.css**
```css
.App {
  text-align: center;
}

.App-header {
  background-color: #282c34;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-size: calc(10px + 2vmin);
  color: white;
}
```

**package.json**
```json
{
  "name": "aws-cicd-react-demo",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": [
      "react-app"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
```

## 2. CI/CD設定ファイル

**buildspec.yml** (リポジトリのルートに配置)
```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 16
  pre_build:
    commands:
      - echo Installing dependencies...
      - npm install
  build:
    commands:
      - echo Building the React application...
      - npm run build
  post_build:
    commands:
      - echo Build completed on `date`

artifacts:
  files:
    - build/**/*
    - appspec.yml
    - scripts/**/*
  discard-paths: no
```

**appspec.yml** (リポジトリのルートに配置)
```yaml
version: 0.0
os: linux
files:
  - source: build
    destination: /var/www/html
hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_application.sh
      timeout: 300
      runas: root
```

**scripts/before_install.sh**
```bash
#!/bin/bash
sudo apt-get update
sudo apt-get -y install apache2
sudo systemctl start apache2
sudo systemctl enable apache2
sudo rm -rf /var/www/html/*
```

**scripts/after_install.sh**
```bash
#!/bin/bash
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

**scripts/start_application.sh**
```bash
#!/bin/bash
sudo systemctl restart apache2
```

## 3. CloudFormationテンプレート

**infra-template.yml** (リポジトリのルートに配置)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'インフラストラクチャ for React Web Application'

Parameters:
  EnvironmentName:
    Description: 環境名（リソース名の接頭辞）
    Type: String
    Default: Dev
  VpcCIDR:
    Description: VPCのCIDRブロック
    Type: String
    Default: 10.0.0.0/16
  PublicSubnet1CIDR:
    Description: パブリックサブネット1のCIDRブロック
    Type: String
    Default: 10.0.1.0/24
  PublicSubnet2CIDR:
    Description: パブリックサブネット2のCIDRブロック
    Type: String
    Default: 10.0.2.0/24
  PrivateSubnet1CIDR:
    Description: プライベートサブネット1のCIDRブロック
    Type: String
    Default: 10.0.3.0/24
  PrivateSubnet2CIDR:
    Description: プライベートサブネット2のCIDRブロック
    Type: String
    Default: 10.0.4.0/24
  InstanceType:
    Description: EC2インスタンスタイプ
    Type: String
    Default: t2.micro
  SSHLocation:
    Description: SSHアクセスを許可するIP範囲
    Type: String
    Default: 0.0.0.0/0
    AllowedPattern: (\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-VPC

  # インターネットゲートウェイ
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-IGW

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  # パブリックサブネット
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Public Subnet 1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Public Subnet 2

  # プライベートサブネット
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet1CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Private Subnet 1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet2CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Private Subnet 2

  # NAT Gateway
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-NAT

  # ルートテーブル（パブリック）
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Public Routes

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  # ルートテーブル（プライベート）
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName} Private Routes

  DefaultPrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnet1

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnet2

  # セキュリティグループ
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB用セキュリティグループ
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ALB-SG

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Webサーバー用セキュリティグループ
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref SSHLocation
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-WebServer-SG

  # EC2インスタンス用IAMロール
  WebServerRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforAWSCodeDeploy
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-WebServer-Role

  WebServerInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref WebServerRole

  # EC2インスタンス
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      SubnetId: !Ref PrivateSubnet1
      ImageId: ami-07c589821f2b353aa # Amazon Linux 2 AMI (リージョンに合わせて変更してください)
      IamInstanceProfile: !Ref WebServerInstanceProfile
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum update -y
          yum install -y ruby
          yum install -y aws-cli
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          
          # CodeDeployエージェントのインストール
          cd /home/ec2-user
          aws s3 cp s3://aws-codedeploy-${AWS::Region}/latest/install . --region ${AWS::Region}
          chmod +x ./install
          ./install auto
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-WebServer

  # ALB
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ALB

  ALBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 30
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 5
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: instance
      Targets:
        - Id: !Ref WebServerInstance
          Port: 80
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-TargetGroup

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ALBTargetGroup
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP

Outputs:
  VPC:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VPC-ID

  ALBDNSName:
    Description: ALBのDNS名
    Value: !GetAtt ApplicationLoadBalancer.DNSName
    Export:
      Name: !Sub ${EnvironmentName}-ALB-DNS

  WebServerInstance:
    Description: EC2インスタンスID
    Value: !Ref WebServerInstance
    Export:
      Name: !Sub ${EnvironmentName}-EC2-Instance
```

## 4. マネジメントコンソールでCI/CDパイプラインを設定する手順

### ステップ1: CloudFormationスタックをデプロイ

1. AWSマネジメントコンソールにログイン
2. CloudFormationサービスに移動
3. 「スタックの作成」→「新しいリソースを使用」を選択
4. 「テンプレートの指定」画面で「テンプレートファイルのアップロード」を選択し、上記の`infra-template.yml`をアップロード
5. スタック名とパラメータを設定し、「次へ」
6. 必要に応じてタグやIAM権限を設定し、「次へ」
7. 設定を確認し、「スタックの作成」をクリック

### ステップ2: CodeDeployアプリケーションを作成

1. CodeDeployサービスに移動
2. 「アプリケーションの作成」をクリック
3. アプリケーション名（例: `react-app`）を入力
4. コンピューティングプラットフォームとして「EC2/オンプレミス」を選択
5. 「作成」をクリック
6. アプリケーションが作成されたら、「デプロイグループの作成」をクリック
7. デプロイグループ名（例: `react-app-deploy-group`）を入力
8. サービスロールは`AWSCodeDeployRole`ポリシーがアタッチされたロールを選択
9. 「インプレース」デプロイタイプを選択
10. 環境設定で「Amazon EC2インスタンス」を選択し、タグキー`Name`、値に先ほど作成したEC2インスタンス名を指定
11. デプロイ設定は「CodeDeployDefault.AllAtOnce」を選択
12. ロードバランサーを有効にし、先ほど作成したターゲットグループを選択
13. 「作成」をクリック

### ステップ3: S3バケットを作成（ビルド成果物用）

1. S3サービスに移動
2. 「バケットを作成」をクリック
3. バケット名（例: `react-app-artifacts-123456`）を入力（一意である必要あり）
4. リージョンを選択し、デフォルト設定のまま「バケットを作成」をクリック

### ステップ4: CodeBuildプロジェクトを作成

1. CodeBuildサービスに移動
2. 「ビルドプロジェクトを作成する」をクリック
3. プロジェクト名（例: `react-app-build`）を入力
4. Oriject typeは「Default Project」を選択
5. ソースプロバイダーとして「GitHub」を選択
6. リポジトリの場所として「自分の GitHub アカウントのリポジトリ」を選択
7. GitHubに接続し、リポジトリを選択. イベントトリガーを無効にする（CodePipelineから呼び出すため）
8. 環境イメージとして「マネージド型イメージ」を選択
9. オペレーティングシステム「Amazon Linux 2」、ランタイム「Standard」、イメージ「aws/codebuild/amazonlinux2-x86_64-standard:3.0」を選択
10. サービスロールは新しいサービスロールを作成するか既存のロールを使用
11. ビルド仕様は「buildspecファイルを使用する」を選択
12. アーティファクトの設定：
   - タイプ: 「Amazon S3」
   - バケット名: 先ほど作成したS3バケットを選択
   - パス接頭辞: `build/`（オプション）
   - アーティファクトの名前: `ReactAppBuild`
   - アーティファクトのパッケージ化: 「ZIP」
13. 「ビルドプロジェクトの作成」をクリック

### ステップ5: CodePipelineを作成

1. CodePipelineサービスに移動
2. 「パイプラインを作成する」をクリック
3. パイプライン名（例: `react-app-pipeline`）を入力
4. サービスロールは新しいサービスロールを作成
5. 「次へ」をクリック
6. ソースステージ:
   - ソースプロバイダー: 「GitHub（バージョン2）」
   - GitHub に接続して、リポジトリとブランチ（main）を選択
   - 検出オプション: 「GitHub ウェブフック (推奨)」
   - 「次へ」をクリック
7. ビルドステージ:
   - ビルドプロバイダー: 「AWS CodeBuild」
   - リージョン: 現在のリージョン
   - プロジェクト名: 先ほど作成したCodeBuildプロジェクトを選択
   - 「次へ」をクリック
8. デプロイステージ:
   - デプロイプロバイダー: 「AWS CodeDeploy」
   - リージョン: 現在のリージョン
   - アプリケーション名: 先ほど作成したCodeDeployアプリケーションを選択
   - デプロイグループ: 先ほど作成したデプロイグループを選択
   - 「次へ」をクリック
9. 設定を確認し、「パイプラインの作成」をクリック

## 5. 動作確認

1. GitHubリポジトリに変更をpushします
2. CodePipelineが自動的に開始され、各ステージ（ソース→ビルド→デプロイ）が実行されます
3. デプロイが完了したら、CloudFormationのアウトプットに表示されているALBのDNS名にアクセスして、Reactアプリケーションが表示されることを確認します

## まとめ

以上の手順で、GitHubのReactアプリケーションをEC2+ALB構成に自動デプロイするCI/CDパイプラインが構築できます。CodePipelineは、GitHubのmainブランチへのpushをトリガーに起動し、CodeBuildでアプリケーションをビルドし、CodeDeployでEC2インスタンスにデプロイします。

このパイプラインにより、コードの変更が自動的にテスト・デプロイされ、アプリケーションの継続的な提供が可能になります。