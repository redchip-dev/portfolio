# Cloud Run 배포 안내

이 안내는 명령 준비용입니다. 아직 Cloud Run에 배포하거나 실제 API를 호출하지 않았습니다.

## 현재 사용자 보고 기준 준비 상태

프로젝트: project-da922ce9-0d65-4f78-8bb

런타임 서비스 계정: web-planning-runtime@project-da922ce9-0d65-4f78-8bb.iam.gserviceaccount.com

| Secret Manager 이름 | 서버 환경변수 |
|---|---|
| gemini-api-key | GEMINI_API_KEY |
| openai-api-key | OPENAI_API_KEY |
| claude-api-key | ANTHROPIC_API_KEY |

사용자가 세 Secret 저장과 런타임 서비스 계정에 대한 각 Secret의 Secret Manager Secret Accessor 권한 부여를 완료했다고 보고했습니다. 실제 권한·Secret 버전은 아직 서버에서 확인하지 않았습니다.

## 배포 전 결정

1. 각 공급자에서 계정이 이용 가능한 모델 ID를 확인합니다.
2. 세 공급자의 API 잔액·결제 상태·사용 한도를 확인합니다.
3. Cloud Run, Cloud Build, Artifact Registry API와 소스 배포자의 권한을 확인합니다. 런타임 계정에는 빌드용 광범위 권한을 추가하지 마세요. 배포자/빌드 계정 권한과 런타임 권한은 별개입니다.

## Cloud Shell에서 ZIP 준비

Google Cloud Console의 Cloud Shell을 열고 파일 업로드 메뉴로 web-planning-server.zip을 업로드하세요.

```sh
unzip web-planning-server.zip
cd web-planning-server
npm test
gcloud config set project project-da922ce9-0d65-4f78-8bb
```

## 비공개 서비스 배포 명령

아래 세 CHOOSE_* 값을 실제 사용 가능한 모델 ID로 바꾼 다음에만 실행하세요. 소스 빌드와 서비스 생성으로 클라우드 비용이 발생할 수 있습니다. 서울 리전은 제안값입니다.

```sh
gcloud run deploy web-planning-api \
  --source . \
  --region asia-northeast3 \
  --service-account web-planning-runtime@project-da922ce9-0d65-4f78-8bb.iam.gserviceaccount.com \
  --no-allow-unauthenticated \
  --set-secrets 'GEMINI_API_KEY=gemini-api-key:1,OPENAI_API_KEY=openai-api-key:1,ANTHROPIC_API_KEY=claude-api-key:1' \
  --set-env-vars 'GEMINI_MODEL=CHOOSE_GEMINI_MODEL,OPENAI_MODEL=CHOOSE_OPENAI_MODEL,CLAUDE_MODEL=CHOOSE_CLAUDE_MODEL,MAX_OUTPUT_TOKENS=4096' \
  --min-instances 0 \
  --max-instances 1 \
  --concurrency 1 \
  --timeout 300 \
  --memory 512Mi
```

Secret 버전 1이 활성화된 값인지 확인하고, 다른 버전이면 명령에서 수정하세요. 버전을 고정하므로 키 변경 후에는 새 Secret 버전으로 다시 배포해야 합니다. Cloud Run이 비공개이므로 호출 계정에 Cloud Run Invoker 권한이 필요합니다. 서버 자체에는 별도의 로그인 UI나 인증 미들웨어가 없습니다. 공개 서비스로 바꾸지 마세요. 인스턴스 제한은 비용 총액을 보장하지 않습니다.

## 유료 AI 호출 없이 확인

배포된 서비스는 인증 프록시로 로컬에서 확인할 수 있습니다. 다음 명령을 실행한 터미널은 열어 두세요.

```sh
gcloud run services proxy web-planning-api --region asia-northeast3 --port 8080
```

별도 터미널에서:

```sh
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/readyz
```

ready:true는 환경변수가 준비되었다는 뜻이며 키 인증·모델 권한·잔액 확인은 아닙니다.

## 실제 AI 1회 시험

계정별 API 과금과 모델을 확인한 뒤 진행하세요. 아래 실행은 Gemini, OpenAI, Claude에 각각 한 번 요청하며 실패하면 다음 단계를 호출하지 않습니다.

```sh
curl --fail-with-body http://127.0.0.1:8080/api/plan \
  -H 'Content-Type: application/json' \
  --data '{"task":"소규모 매장 예약 사이트를 기획해줘","references":"30분 단위 예약. 운영자가 예약을 승인한다."}' \
  --output result.json
```

성공한 result.json을 Markdown으로 변환합니다. 실패 응답에는 markdown이 없으므로 먼저 HTTP 성공 여부를 확인하세요.

```sh
node --input-type=module -e 'import fs from "node:fs"; const r=JSON.parse(fs.readFileSync("result.json","utf8")); if(typeof r.markdown!=="string") throw Error("No Markdown result"); fs.writeFileSync("planning-draft.md",r.markdown);'
```

planning-draft.md를 사람이 검토·수정한 뒤 GitHub 문서 폴더에 저장합니다. .env, ZIP, JS, Dockerfile과 키는 GitHub에 올리지 않습니다.
