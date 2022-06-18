---
title: "Server status 파악을 위한 Slack 알람 추가"
date: 2022-06-19
# classes: wide
toc: true
toc_sticky: true
categories:
  - server
tags:
  - slack
  - service
---

## 상황
Linux에 서비스로 등록하여 동작 중인 서버가 어떤 이유로 동작이 중지되고, restart 주기로 계속 재시작되고 있었는데 이에 대한 파악이 늦었다.
트래픽이 많지도 않고 서비스가 크리티컬하지도 않지만, 이러한 상황을 방지하고자 slack에 간단히 에러로그와 함께 서버가 종료됨을 알리는 알람을 추가하고 싶었다.

유사한 상황을 구현한 예시가 있어([예시]) 이를 따라서 적용하였다. 여기서는 [slacktee]를 사용하였다.
[slacktee]는 linux의 tee command와 동일하게 사용가능하게 만든 스크립트이다. tee command가 입력을 file에 썼다면, slacktee는 입력을 slack에 보낸다.

<p align="center">
  <img src="https://github.com/course-hero/slacktee/raw/slacktee-readme-images/slacktee_demo.gif" />
</p>
<p style="text-align: center; bold">
  <em>
    slacktee의 
      <a href="https://github.com/coursehero/slacktee">
        예시
      </a>
  </em>
</p>

## 과정
### 1. Authentication Token 생성(Slack에 bots 추가 (또는) Slack app 추가)
[slacktee의 configuration](https://github.com/coursehero/slacktee#configuration)에 따르면 두 가지 방법으로 notification을 채널에 띄울 수 있는 authentication token을 생성할 수 있다.

Slack에서는 slack app 생성을 통한 token 생성을 권장하나, 여기서는 편의를 위해 slack Bot을 사용해 token을 생성하였다. 간단히, [link](https://slack.com/apps/A0F7YS25R-bots)에서 install을 수행하면 된다.

(**[참고]** slack bot 페이지의 설명과 같이 해당 방식은 legacy이므로, 향 후 deprecated 가능성이 있다. 따라서, 더 안정적인 환경을 선호한다면 slack app을 추가하여 token 생성을 권장한다. 이는 [slacktee의 configuration](https://github.com/coursehero/slacktee#configuration)을 따라가면 된다.)

### 2. Server에 slacktee 설치
```
# Clone git repository
git clone https://github.com/course-hero/slacktee.git

# Install slacktee.sh
bash ./slacktee/install.sh
```

셋업 시에 나오는 항목을 채워준다.

```
token=""            # The authentication token of the bot user. Used for accessing Slack APIs.
channel=""          # Default channel to post messages. '#' is prepended, if it doesn't start with '#' or '@'.
tmp_dir="/tmp"      # Temporary file is created in this directory.
username="slacktee" # Default username to post messages.
icon="ghost"        # Default emoji or a direct url to an image to post messages. You don't have to wrap emoji with ':'. See http://www.emoji-cheat-sheet.com.
attachment=""       # Default color of the attachments. If an empty string is specified, the attachments are not used.
```

아래 명령어로 간단하게 테스트가 가능하다.
```
echo 'testing' | /usr/local/bin/slacktee
```

### 3. Service에 slacktee 항목 추가
서버가 종료되었을 때의 log를 함께 slack에서 확인하고자, 종료 시에는 log를 출력하도록 추가하였다.

서비스 명이 my-service일 때, journalctl을 활용하여 가장 최근 30줄을 slacktee를 통하여 slack으로 전송한다.
```
[Service]
ExecStartPre=-/bin/sh -c "echo 'My Server started' | /usr/local/bin/slacktee.sh"

ExecStopPost=/bin/sh -c "{ echo 'My server stopped, last lines from logs:'; journalctl -u my-service -n 30 --no-pager; } | /usr/local/bin/slacktee.sh"
```
<p align="center">
<a href="https://ibb.co/DVPt5Y1"><img src="https://i.ibb.co/YR43pXN/Screen-Shot-2022-06-19-at-12-47-21-AM.png" alt="Screen-Shot-2022-06-19-at-12-47-21-AM" border="0" width="1000"></a>
</p>
<p style="text-align: center; bold"> <em>결과 예시</em> </p>

## Reference
 1. <https://www.scaledrone.com/blog/real-time-notifications-from-systemd-to-slack/>
 2. <https://github.com/coursehero/slacktee>

[예시]: https://www.scaledrone.com/blog/real-time-notifications-from-systemd-to-slack/
[slacktee]: https://github.com/coursehero/slacktee