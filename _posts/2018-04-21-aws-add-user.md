---
layout: post
title: "AWS 계정 생성 스크립트"
description: "AWS 계정을 로컬에서 스크립트로 생성해봅니다."
date: 2018-04-21
tags: [AWS, AWS-CLI, IAM, 계정생성]
comments: true
share: true
---

콘솔에서 매번 계정 생성하기 귀찮아서 간단하게 스크립트을 작성해보았다.

작업 순서는 아래와 같다.

* AWS CLI을 실행할 수 있도록 각 계정에 맞춰서 .aws/config 와 credentials 을 설정한다.
  1. .aws/config
  2. .aws/credentials
* 로컬에서 AWS CLI을 사용하려면 MFA을 해야하므로 해당 스크립크를 생성한다.
* AWS 계정을 생성하는 스크립트를 생성한다.
* 정리하면, aws-cli.sh로 MFA 진행하여 AWS CLI을 사용할 수 있도록 하고, aws-user-add.sh로 계정을 생성하면 끗.
  1. 계정 생성 시 초기 암호는 1234로 임시로 설정하고 최초 로그인 시 암호를 변경하는 것으로 작업
  2. 그리고 각 AWS 계정의 그룹 리스트를 확인하고 그룹을 선택

.aws/config와 .aws/credentials은 환경에 맞춰서 알아서..

# MFA 인증하는 스크립트

```sh
#!/bin/bash

set -e

read -p 'Enter AWS_PROFILE: ' AWS_PROFILE

caller_identity=($(aws --profile "$AWS_PROFILE" sts get-caller-identity --output text))

AWS_ACCOUNT_NUMBER="${caller_identity[0]}"
AWS_IAM_USER_ARN="${caller_identity[1]}"
AWS_IAM_USERNAME="$(basename "$AWS_IAM_USER_ARN")"
MFA_SERIAL="arn:aws:iam::$AWS_ACCOUNT_NUMBER:mfa/$AWS_IAM_USERNAME"

echo "AWS Account number: $AWS_ACCOUNT_NUMBER"
echo "IAM Username: $AWS_IAM_USERNAME"
echo "MFA Serial: $MFA_SERIAL"

read -p 'Enter MFA code: ' OTP_TOKEN

session_token=($(aws --profile "$AWS_PROFILE" sts get-session-token --serial-number $MFA_SERIAL --token-code $OTP_TOKEN --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --output text))

export AWS_ACCESS_KEY_ID="${session_token[0]}" AWS_SECRET_ACCESS_KEY="${session_token[1]}" AWS_SESSION_TOKEN="${session_token[2]}" AWS_PROFILE=$AWS_PROFILE

aws sts get-caller-identity

exec $SHELL -l
```

# 계정 생성하는 스크립트

```sh
#!/bin/bash

clear

echo ==================================================================
echo "                AWS User Add Process Start                      "
echo ==================================================================

echo -n "useremail : "
read useremail
aws iam create-user --user-name $useremail

echo ==================================================================
echo "           AWS User PassWord Process Start                      "
echo ==================================================================

aws iam create-login-profile --user-name $useremail --password 1234 --password-reset-required

echo ==================================================================
echo "              AWS User Group Process Start                      "
echo ==================================================================

aws iam list-groups | grep GroupName

echo -n "group : "
read group

aws iam add-user-to-group --user-name $useremail --group-name $group
```