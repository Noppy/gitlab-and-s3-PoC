# Glitlab + S3検証
# 検証概要



# 作成手順
## (1)事前設定
### (1)-(a) 作業環境の準備
下記を準備します。もしCLI実行環境がない場合は、次の(1)-(b)を参照し、Cloud9の環境を準備して実行します。
* bashが利用可能な環境(LinuxやMacの環境)
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定

### (1)-(b) (Option) Cloud9環境準備
Cloud9利用のためには、インターネット経由でCloud9用の絵c2インスタンスにhttpsでアクセス可能である必要があります。
#### (i) Cloud9インスタンスの作成
+ マネージメントコンソールに、AdministratorAccess権限のあるユーザでログインします。
+ サービスから<code>AWS Cloud9</code>に移動します。
+ <code>Create environment</code>ボタンを押します。
+ Name environment
  + NameとDescriptionに任意の情報を入れます。
+ Environment settings  *デフォルトのままでOKですが、念の為記載します。
  + Environment type: <code>Create a new EC2 instance for environment (direct access)</code>を選択
  + Instance type: <code>t2.micro (1 GiB RAM + 1 vCPU)</code>を選択
  + Platform: <code>Amazon Linux</code>
+ Network settings (advanced):
  + デフォルトでは、デフォルトVPCにデプロイされます。基本そのままにします。
+ 設定内容確認
  + 設定内容を確認し、<code>Create environment</code>を実行。
#### (ii) Cloud9環境のアクセスと環境の確認
インスタンスが作成されると、下記のようなCloud9の操作画面がブラウザに表示されます。
以後は、右下のコマンドラインで作業を行います。(画面が小さい場合はコマンドラインを上に拡大することが可能です。)

![Cloud9画面](./Documents/02_cloud9.png)

#### (iii) AWS CLIの実行確認 
コマンドラインで、aws CLIが利用できることを確認します。stsコマンドで、実行に利用するセッション情報が表示できることを確認します。以下の形式でCloud9のインスタンスを作成したユーザの権限情報が表示されれば成功です。

```shell
aws sts get-caller-identity
{
    "UserId": "XXXXXXX:XXXXXX",
    "Account": "999999999999",
    "Arn": "arn:aws:sts::999999999999:XXXXXXX/XXXXXX"
}
```
### (1)-(b) 環境作成用ファイルのClone
この構築用のファイル一式を実行環境で展開します。

#### (i) git clone
下記コマンドで検証用資材をgit cloneします。
```shell
git clone https://github.com/Noppy/gitlab-and-s3-PoC.git
```
### (1)-(c) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=default  #デフォルト以外のプロファイルの場合は、利用したいプロファイル名を指定
export REGION=$(aws --profile ${PROFILE} configure get region)
echo "${PROFILE}  ${REGION}"
```

## (2)ネットワーク環境の作成(CloudFormation利用)
gitコマンド用のクライアントVPC(ClientVPC)、gitlab用VPC(GitlabVPC)を作成しTransit Gatewayで接続します。(VPC接続は、VPC Peeringでも可能)
なお、CloudFormationの進捗状況は、別途マネージメントコンソールの画面をだしCloudFormationのスタックを表示するとわかりやすいです。
<img src="./Documents/" whdth=500>

### (2)-(a) ClientVPC/GitlabVPC作成
```shell
#ClientVPC
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcInternalDnsNameEnable",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "ClientVPC"
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.1.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1Name",
    "ParameterValue": "PubSub1"
  },
  {
    "ParameterKey": "PublicSubnet1Cidr",
    "ParameterValue": "10.1.0.0/19"
  },
  {
    "ParameterKey": "PublicSubnet2Name",
    "ParameterValue": "PubSub2"
  },
  {
    "ParameterKey": "PublicSubnet2Cidr",
    "ParameterValue": "10.1.32.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet1Name",
    "ParameterValue": "PrivateSub1"
  },
  {
    "ParameterKey": "PrivateSubnet1Cidr",
    "ParameterValue": "10.1.128.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet2Name",
    "ParameterValue": "PrivateSub2"
  },
  {
    "ParameterKey": "PrivateSubnet2Cidr",
    "ParameterValue": "10.1.160.0/19"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name GitlabS3PoC-ClientVPC \
    --template-body "file://./cfns/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;

# GitlabVPC
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcInternalDnsNameEnable",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "GitlabVPC"
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.2.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1Name",
    "ParameterValue": "TgwSub1"
  },
  {
    "ParameterKey": "PublicSubnet1Cidr",
    "ParameterValue": "10.2.0.0/19"
  },
  {
    "ParameterKey": "PublicSubnet2Name",
    "ParameterValue": "TgwSub2"
  },
  {
    "ParameterKey": "PublicSubnet2Cidr",
    "ParameterValue": "10.2.32.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet1Name",
    "ParameterValue": "gitlabSub1"
  },
  {
    "ParameterKey": "PrivateSubnet1Cidr",
    "ParameterValue": "10.2.128.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet2Name",
    "ParameterValue": "gitlabSub2"
  },
  {
    "ParameterKey": "PrivateSubnet2Cidr",
    "ParameterValue": "10.2.160.0/19"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name GitlabS3PoC-GitlabVPC  \
    --template-body "file://./cfns/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```
### (2)-(b) TransitGateway接続(CloudFormation利用)
![TransitGateway](./Documents/)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-TGW \
    --template-body "file://./cfns/tgw.yaml" ;
```



```
## (4)VPCE作成(CloudFormation利用)
![TransitGateway](./Documents/14_VPCE.png)
### (4)-(a) VPCE(PrivateLink)
```shell
# Internal-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-InternalVPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "InternalVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Internal-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;

# DMZ-Inbound-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-DMZ-Inbound-VPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZInboundVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Inbound-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;

# DMZ-Outbound-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-DMZ-Outbound-VPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZOutboundVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Outbound-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;
```
### (4)-(b) VPCE(S3)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-VPCES3 \
    --template-body "file://./cfn/vpce_s3.yaml" ;
```
## (5)SecurityGroup作成(CloudFormation利用)
EC2インスタンスに適用するSecurityGroupを作成します。
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-SecurityGroups \
    --template-body "file://./cfn/sg.yaml" ;
```
## (6) バッチ・リレーメールインスタンスの準備
バッチ用インスタンス、およびリレーメール用インスタンスの共通設定を行います。
### (6)-(a) インスタンスロール作成 (CloudFormation利用)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-IAM-For-Instance \
    --template-body "file://./cfn/iam-relaymail.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```
### (6)-(b)共通のパラメータ設定 
ここで必ず、検証で外部からSubmissionPortにメール送信する時に利用する環境のパブリックIPを<code>NETWORK_FOR_SEMD_MAIL</code>に指定してください。また、利用するキーペアを<code>KEYNAME</code>に設定して下さい。
```shell
NETWORK_FOR_SEMD_MAIL="<<検証時の外部クライアントのIPをCIDRで記載>>"
POSTFIX_MYNETWORK="127.0.0.1, 10.0.0.0/8"
POSTFIX_MYNETWORK_INBOUND="${POSTFIX_MYNETWORK},${NETWORK_FOR_SEMD_MAIL}"

KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。  

#最新のAmazon Linux2のAMI IDを取得します。
AL2_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;
echo -e "POSTFIX_MYNETWORK_INBOUND=${POSTFIX_MYNETWORK_INBOUND}\nKEYNAME=${KEYNAME}\nAL2_AMIID=${AL2_AMIID}"
```
## (7)リレーメールインスタンス
### (6)-(a) (Option)Gmail接続用のパスワードのStore
検証でGmailに接続するためのユーザIDとパスワードを安全に管理するため、System ManagerのParameter storeに格納します。<code>GMAIL_SASL_INFO</code>にPostfixで定義する形式でGmailの認証情報を設定します。Gmailの認証はユーザとパスワードですが、パスワードはアプリケーション用のパスワードをgoogleで払い出して設定するようにします。
![SSM Parameter Store](./Documents/16_SSM_ParameterStore.png)
```shell
# GMAILの接続用の認証情報の設定
# username@gmail:passwordに、アカウント名とgoogleで払い出したアプリパスワードを設定します。
GMAIL_SASL_INFO="[smtp.gmail.com]:587 username@gmail:password"

# Parameter storeのPut
aws --profile ${PROFILE} \
  ssm put-parameter \
    --name "mail-poc-gmail-sals-info" \
    --value "${GMAIL_SASL_INFO}" \
    --type SecureString \
    --tags "Key=Name,Value=mail-poc-gmail-sals-info"
```

### (7)-(b) OutboundのRelayMailインスタンス作成 (CloudFormation利用)
![SSM Parameter Store](./Documents/17_OutboundInstance.png)
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  },
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "PostfixMynetwork",
    "ParameterValue": "'"${POSTFIX_MYNETWORK}"'"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Outbound-RelayMail-Instance \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/relaymail-outbound.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```

## (8) バッチインスタンス作成(CloudFormation利用)
![SSM Parameter Store](./Documents/18_BatchInstance.png)
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  },
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "PostfixMynetwork",
    "ParameterValue": "'"${POSTFIX_MYNETWORK}"'"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Batch-Instance \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/batch.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```
## (9) InboundのRelayMailインスタンス作成 (CloudFormation利用)
![SSM Parameter Store](./Documents/19_InboundInstance.png)
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  },
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "PostfixMynetwork",
    "ParameterValue": "'"${POSTFIX_MYNETWORK_INBOUND}"'"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Inbound-RelayMail-Instance \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/relaymail-inbound.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```


## (10) テスト
### (10)-(a)メール送信テスト
バッチインスタンスにSystems Manager Session Managerでアクセスし、メール送信のテストをします。
#### (i)Systems Manager Session Managerによるバッチインスタンスアクセス
+ マネージメントコンソールで、Systems Managerのサービスを開く
+ 左側のナビゲーションペインから<code>セッションマネージャー</code>を開く
+ <code>セッションを開始する</code>をクリックし、次の画面でリストから<code>MailPoC-InternalVPC-Batch</code>を選び、右上の<code>セッションを開始する</code>
#### (ii)コマンドによるメール送信
```shell
sudo -i -u ec2-user
To_Address="xxxxxx@xxxxxxx.com" #Gmailからのメール受信が可能なアドレスに変更する

Subject="TestMail-$(date '+%Y%m%d%H%M%S')"
SMTP="smtp=smtp://outbound-relaymail.mailpoc.local:587"
From_Address="xxx"


echo "テストメール$(date '+%Y%m%d%H%M%S')" | mail -s $Subject -S $SMTP -r $From_Address $To_Address
```

### (10)-(b)メール受信テスト
#### (i) Cloud9(構築実行環境)からのメール送信
```shell
#ローカールでのmail送信テスト用にmailコマンドをインストール
sudo yum -y install mailx

#InboundのNLBのURLを取得
NLB_DNS=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name MailPoC-Inbound-RelayMail-Instance \
        --query 'Stacks[].Outputs[?OutputKey==`InboundRelaymailNlbDns`].[OutputValue]')

echo -e "NLB_DNS= $NLB_DNS"


#メール送信
Subject="TestMail-$(date '+%Y%m%d%H%M%S')"
SMTP="smtp=smtp://${NLB_DNS}:587"
From_Address="xxx"
To_Address="ec2-user@mailpoc.pub"

echo "テストメール$(date '+%Y%m%d%H%M%S')" | mail -s $Subject -S $SMTP -r $From_Address $To_Address
```
#### (ii)バッチインスタンスでのメール受信確認
ec2-userにて、mailコマンドを実行しメールが受信されていることを確認します。
```shell
mail
Heirloom Mail version 12.5 7/5/10.  Type ? for help.
"/var/spool/mail/ec2-user": 1 message 1 new
>N  1 xxx                   Mon Sep 14 11:24  21/968   "TestMail-20200914112456"
&
```