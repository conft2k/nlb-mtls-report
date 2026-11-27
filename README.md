# nlb-mtls-report

NLB mutual TLS 테스트 보고서의 GitHub Pages 소스입니다. Pages 설정에서 `main` 브랜치의 루트(`/`)를 지정하면 Actions 없이 빌드됩니다.

## 결과 채우기

`index.md`의 결과 자리는 모두 다음 형태입니다.

```
대기<!-- RESULT: <id> -->
```

`대기`와 주석을 함께 실제 값으로 바꿉니다. 남은 자리 확인:

```bash
grep -o 'RESULT: [A-Za-z0-9-]*' index.md
```

공개 저장소이므로 계정 ID, ARN, 리소스 ID, DNS 이름, 프로파일 이름은 넣기 전에 지웁니다.
