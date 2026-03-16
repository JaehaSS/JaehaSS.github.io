---
layout: post
title: "Github Action Workflow를 직접 작성해서 오픈 소스 기여해보자!"
date: 2025-01-19
description: "VPN 환경에서의 CI/CD 자동화를 위해 FortiClient VPN GitHub Action을 직접 개발하고 Marketplace에 배포한 오픈 소스 기여 경험을 공유합니다."
tags: [오픈 소스]
tistory_id: 57
---

# 들어가며

우리 회사에서는 이번에 처음으로 CI/CD 파이프라인을 구축하게 되었습니다. 그동안 배포는 VPN을 통해 서버에 SSH로 접속하고, 파일질라 같은 툴을 이용해 소스를 직접 다운로드하여 실행하는 방식으로 이루어졌습니다. 이와 같은 수동 배포 방식은 반복적인 작업에서 발생하는 휴먼 에러와 긴 배포 시간으로 인해 큰 문제가 되고 있었습니다.

이러한 문제를 해결하고자, GitHub Action을 활용하여 VPN 환경에서도 안정적이고 자동화된 배포 프로세스를 구현하려고 하였습니다. 그러나 사내에서 사용 중인 VPN 방식과 기존에 GitHub Marketplace에서 제공하는 워크플로우가 맞지 않아, 결국 직접 오픈 소스로 GitHub Action을 개발하게 되었습니다.

이 글에서는 이러한 배경과 함께, 제가 어떻게 직접 VPN Workflow를 했는지 공유하고자 합니다.

# 구현 과정

CLI + 우분투 환경에서 어떻게 FortiVPN를 다운받고 VPN 환경에 접속하는 방법을 알아보고 자동화했습니다.

## 설계 방향

1. FortiVPN에 필요한 Ubuntu lib 다운
2. FortiVPN 다운로드
3. Shell 파일을 통한 FortiVPN Config 설정
4. Shell 파일을 통한 FortiVPN 접속

## Github Action action.yml 작성

Github Marketplace에 소스를 올릴려면 Github Repo에는 **action.yml** 파일이 존재해야합니다. 해당 Yml에는 다양한 속성들이 많으니 [링크](https://docs.github.com/ko/actions/sharing-automations/creating-actions/metadata-syntax-for-github-actions) 참고하시면 됩니다.

```yaml
name : "fortclient-vpn"
description : "Connect to FortiClient VPN"
autor : "JaehaSS"
inputs:
  VPN_IP:
    description: "VPN IP Address"
    required: true
  VPN_PORT:
    description: "VPN Port"
  VPN_USERNAME:
    description: "VPN Username"
    required: true
  VPN_PASS:
    description: "VPN Password"
    required: true

runs:
  using: "composite"
  steps:
    - name: Install Dependencies
      shell: bash
      run: |
        sudo apt-get update
        sudo apt-get install -y libnss3-tools ppp libappindicator1 expect ca-certificates

    - name: Download and Install FortiClient VPN
      shell: bash
      run: |
        wget https://filestore.fortinet.com/forticlient/forticlient_vpn_7.4.0.1636_amd64.deb -O forticlient.deb
        sudo dpkg -i forticlient.deb

    - name: Create VPN Profile
      shell: bash
      run: |
        #!/usr/bin/expect
        echo '#!/usr/bin/expect -f' > create_vpn_profile.exp
        echo 'spawn /opt/forticlient/fortivpn edit myvpn' >> create_vpn_profile.exp
        echo 'expect -re "Type"' >> create_vpn_profile.exp
        echo 'send -- "1\r"' >> create_vpn_profile.exp
        echo 'set timeout -1' >> create_vpn_profile.exp
        echo 'expect -re "Remote Gateway:"' >> create_vpn_profile.exp
        echo 'send -- "${{ inputs.VPN_IP }}\r"' >> create_vpn_profile.exp
        echo 'set timeout -1' >> create_vpn_profile.exp
        echo 'expect -re "Port"' >> create_vpn_profile.exp
        echo 'send -- "${{ inputs.VPN_PORT }}\r"' >> create_vpn_profile.exp
        echo 'set timeout -1' >> create_vpn_profile.exp
        echo 'expect -re "Authentication"' >> create_vpn_profile.exp
        echo 'send -- "3\r"' >> create_vpn_profile.exp
        echo 'set timeout -1' >> create_vpn_profile.exp
        echo 'expect -re "Certificate"' >> create_vpn_profile.exp
        echo 'send -- "3\r"' >> create_vpn_profile.exp
        echo 'set timeout -1' >> create_vpn_profile.exp
        echo 'expect eof' >> create_vpn_profile.exp
        chmod +x create_vpn_profile.exp
        ./create_vpn_profile.exp

    - name: List VPN Script
      shell: bash
      run: |
        /opt/forticlient/fortivpn list
        /opt/forticlient/fortivpn view myvpn

    - name: Create VPN Connect Script
      shell: bash
      run: |
        echo '#!/usr/bin/expect -f' > vpnconnect.exp
        echo 'spawn /opt/forticlient/fortivpn connect myvpn --user=${{ inputs.VPN_USERNAME }} --password' >> vpnconnect.exp
        echo 'expect -exact "password:"' >> vpnconnect.exp
        echo 'send -- "${{ inputs.VPN_PASS }}\r"' >> vpnconnect.exp
        echo 'set timeout -1' >> vpnconnect.exp
        echo 'expect -re "Confirm "' >> vpnconnect.exp
        echo 'send -- "y\r"' >> vpnconnect.exp
        echo 'set timeout -1' >> vpnconnect.exp
        echo 'expect eof' >> vpnconnect.exp
        chmod +x vpnconnect.exp
        ./vpnconnect.exp
```

저는 action.yml을 이렇게 작성했습니다.

## Github Repo 배포

Yml 파일과 Input 속성에 해당하는 값들에 대한 설명을 README.md에 설명 후 배포를 하면 Marketplace에 배포할 수 있다고 나옵니다.

배포 후에는 Marketplace에서 직접 배포한 workflow를 볼 수 있습니다.

# 마치며

평소에 오픈소스를 기여해보거나 직접 만들어보고 싶었습니다. 그 기회를 제가 매 번 미루고 있었는데 이렇게 기회가 직접적으로 제게 과제로 다가와서 오픈 소스를 만들어 보니 정말 기분이 좋았습니다.
