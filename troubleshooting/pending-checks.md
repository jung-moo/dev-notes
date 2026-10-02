
# 추가 확인 사항

트러블슈팅 과정에서 아직 확인하지 못한 사항을 문서별로 정리한다. 확인이 끝나면 체크하고, 확인 결과와 근거를 관련 문서에 반영한다.

## JBoss Request Size

관련 문서: [JBoss Request Size](jboss-request-size.md#추가-확인-사항)

현재 Apache `8082` → mod_jk → AJP 1.3 → JBoss EAP 7.2 구간을 확인했고, JBoss의 `max-post-size` 변경 후 엑셀 업로드 문제가 해결되었다. Apache 앞단의 구조와 `502 Bad Gateway`가 반환된 정확한 과정은 아직 확인하지 않았다.

- [ ] Apache 앞단에 Nginx가 존재하는지 확인한다.
- [ ] Apache 앞단에 별도의 L4/L7 Load Balancer가 존재하는지 확인한다. (`mod_jk`의 Load Balancer Worker와 구분)
- [ ] 외부 요청이 어떤 경로를 통해 Apache `8082`로 전달되는지 확인한다.
- [ ] 당시 Apache `mod_jk` 로그와 JBoss 로그 등을 확인하여, 요청 크기 제한 초과 시 `502 Bad Gateway`가 어느 구간에서 생성되어 사용자에게 반환되었는지 확인한다.
