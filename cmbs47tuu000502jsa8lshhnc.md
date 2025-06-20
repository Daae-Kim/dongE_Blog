---
title: "CI/CD 적용 이후 고민과 Health Check 도입"
seoTitle: "CI/CD 적용 이후 고민과 Health-check 도입"
seoDescription: "CI/CD 도입 후 서버가 온전한지 체크할 수 있는 Health check 결과를 추가적으로 workflow 에 구성하게되었습니다."
datePublished: Wed Jun 11 2025 15:39:12 GMT+0000 (Coordinated Universal Time)
cuid: cmbs47tuu000502jsa8lshhnc
slug: cicd
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1749657044339/654ab4cc-82d7-46fd-8f74-41ad5b1dfd8b.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1749657348006/b176d8bd-047e-47b6-b3ed-b31c42a81ef5.png
tags: cicd, ci-cd, health-check

---

### EEOS 로그인이 안되는데요?

CI/CD 적용 후 슬랙에는 배포 성공 알림이 올라왔고, 당연히 배포가 완료 된줄 알았다. 서버에 접속해보니 테이블 변경 오류로 애플리케이션 실행이 제대로 되지 않은 상태였다.

### 문제 상황

기존 CI/CD workflow 는 배포 성공 및 실패에 따라 슬랙 메세지가 전송되는 로직으로 작성 되어있지만, 여기서 “성공“은 CD 코드가 끝까지 실행되었을때이다.

만약 CD 코드가 모두 실행되고 슬랙 메세지까지 전송되면 action도 성공하지만, 배포 중에 문제(테이블 문제)가 있을때, 실제 서버는 작동하지 않을 수 있는 가능성이 있었다.

따라서 CD 의 “성공“이 우리가 바라는 성공 (= 새 코드로 배포 완료, 서비스 작동 완료) 인지 한번 더 검증하는 과정이 필요하다.

**그럼 배포 후 서비스가 제대로 작동되는지 어떻게 확인할 수 있을까?**

### CD + API Health check

CD 의 로직에 api health check 로직을 추가하여 배포 후 api health check 결과를 함께 받을 수 있도록 하면 된다. 이렇게 하면 실제 서버가 제대로 올라갔는지 배포 메세지와 함께 한꺼번에 확인 할 수 있을 것이다.

### 개선

```bash
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 20 https://dev.eeos.econovation.kr/api/health-check)
```

curl 을 사용해서 health check API 에 요청을 보내고, -w 옵션을 사용해서 “%{http\_code}“ 응답 HTTP 상태코드만 출력한다. (타임아웃은 10초로 적용시 실패의 경우가 많아서 넉넉히 20초로 늘렸다.)

이 결과로 받은 HTTP\_CODE 값이 200이면 성공, time out이면 미실시, 이외의 값은 실패로 메세지가 전송된다.

기존에 deploy → healthcheck → slack\_notify 를 순차적으로 실행하기 위해 deploy 의 step 으로 구성했었는데, 각자의 단계에서 어떤것이 실패했는지 한눈에 보기가 어려웠다.

그래서 각 단계를 job으로 분리하고 needs 키워드를 사용해서 deploy - health\_check - slack\_notify순서로 실행될 수 있도록 재구성 했다.

![github action에서 각 단계의 결과를 한눈에 확인 가능해졌다](https://cdn.hashnode.com/res/hashnode/image/upload/v1749642875548/b812c16f-11fe-44cb-b8b2-b9be3bc2c343.png align="center")

job 분리후 다음과 같이 단계별로 상태를 한눈에 볼 수 있게되었다.

```bash
name: CD Deploy

jobs:
  deploy:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: SSH 접속 및 배포 실행
        id: deploy
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USERNAME }}
          password: ${{ secrets.DEV_PASSWORD }}
          port: ${{ secrets.DEV_SSH_PORT }}
          script: |
            # 생략

  health_check:
    needs: deploy
    runs-on: ubuntu-latest
    outputs:
      status: ${{ steps.check.outputs.status }}
    steps:
      - name: API Health Check
        id: check
        run: |
          echo "Waiting for container to start..."
          sleep 20
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://dev.eeos.econovation.kr/api/health-check)
          if [ $? -ne 0 ]; then
            echo "Health Check timed out."
            echo "status=timeout" >> $GITHUB_OUTPUT
          elif [ "$HTTP_CODE" = "200" ]; then
            echo "status=success" >> $GITHUB_OUTPUT
          else
            echo "status=failure" >> $GITHUB_OUTPUT
          fi

  slack_notify:
    needs: [deploy, health_check]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Slack 알림 전송
        env:
          #생략
          HEALTH_STATUS: ${{ needs.health_check.outputs.status }}
        run: |
          if [ "${DEPLOY_STATUS}" = "success" ]; then
            DEPLOY_TEXT="성공"
            BASE_COLOR="#36a64f"
            if [ "${HEALTH_STATUS}" = "success" ]; then
              HEALTH_TEXT="성공"
            elif [ "${HEALTH_STATUS}" = "failure" ]; then
              HEALTH_TEXT="실패"
              BASE_COLOR="#FF0000"
            elif [ "${HEALTH_STATUS}" = "timeout" ]; then
              HEALTH_TEXT="Timeout"
              BASE_COLOR="#FFA500"
            else
              HEALTH_TEXT="미실시"
            fi
          else
            DEPLOY_TEXT="실패"
            BASE_COLOR="#FF0000"
            HEALTH_TEXT="미실시"
          fi
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1749646443928/7569aa50-53e3-4653-b3fb-1995a8bfa7c3.png align="center")

healthCheck 결과를 함께 볼 수 있게되어 배포 후 사이트를 추가로 확인 하는 과정이 줄었다 :)

우리 팀내에서는 모니터링 툴을 따로 사용하지 않고 있어서 healthCheck 를 사용했지만, Prometheus과 같은 툴도 좋은 선택이라고 한다. CD를 적용하기 전까지는 healthCheck와 같은 배포 외에 다른 역할을 workflow 에 추가하게 될 줄은 몰랐다. 자동화 배포뿐만 아니라 배포 과정에서 미처 인식 하지 못했었던 불편함을 함께 해결하게되어 더욱 뿌듯한 마음이 들었던 경험이다.