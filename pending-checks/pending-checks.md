
# 추가 확인 사항

트러블슈팅 과정에서 아직 확인하지 못한 사항을 문서별로 정리한다. 확인이 끝나면 체크하고, 확인 결과와 근거를 관련 문서에 반영한다.

## JBoss Request Size

관련 문서: [[jboss-request-size#추가 확인 사항]]

현재 Apache `8082` → mod_jk → AJP 1.3 → JBoss EAP 7.2 구간을 확인했고, JBoss의 `max-post-size` 변경 후 엑셀 업로드 문제가 해결되었다. Apache 앞단의 구조와 `502 Bad Gateway`가 반환된 정확한 과정은 아직 확인하지 않았다.

- [ ] Apache 앞단에 Nginx가 존재하는지 확인한다.
- [ ] Apache 앞단에 별도의 L4/L7 Load Balancer가 존재하는지 확인한다. (`mod_jk`의 Load Balancer Worker와 구분)
- [ ] 외부 요청이 어떤 경로를 통해 Apache `8082`로 전달되는지 확인한다.
- [ ] 당시 Apache `mod_jk` 로그와 JBoss 로그 등을 확인하여, 요청 크기 제한 초과 시 `502 Bad Gateway`가 어느 구간에서 생성되어 사용자에게 반환되었는지 확인한다.

## Jenkins JBoss Deploy Timeout

관련 문서: [[jenkins-jboss-deploy-timeout#추가로 확인할 내용]]

잔존 JBoss 프로세스를 종료한 뒤 재배포에 성공했지만, 최초에 정상 종료되지 않은 근본 원인은 아직 확인하지 못했다. 동일한 문제가 다시 발생하면 다음 항목을 조사한다.

- [ ] JBoss 종료 시점의 서버 로그를 확인한다.
- [ ] 종료 스크립트의 실행 결과를 확인한다.
- [ ] 종료 과정에서 장시간 수행되는 작업을 확인한다.
- [ ] 프로세스가 종료되지 못하도록 하는 Thread가 존재하는지 확인한다.
- [ ] Jenkins가 실행하는 재기동 스크립트의 동작을 확인한다.
- [ ] Jenkins SSH Timeout 설정과 실제 재기동 소요 시간을 확인한다.
