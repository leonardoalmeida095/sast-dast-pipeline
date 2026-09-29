# POC SAST e DAST

Esta prova de conceito exercita um pipeline de segurança em cima de duas aplicações pequenas: um frontend e uma API. O código em si é um eco do navegador. O que a POC valida é o caminho do Bandit no GitHub Actions e do ZAP como job no Cloud Run, com deploy privado e rollback quando o DAST reprova.

## Objetivo

Mostrar um fluxo contínuo em que:

1. o GitHub Actions autentica no GCP por Workload Identity Federation;
2. o Bandit analisa o Python e interrompe o pipeline se houver achado de severidade média ou alta;
3. o frontend e a API sobem como serviços no Cloud Run;
4. o ZAP roda como job, varre a URL pública do load balancer e grava o relatório no Cloud Storage;
5. alerta de risco alto reprova a versão e o tráfego volta para a revisão anterior.

O frontend e a API existem para o ZAP ter o que percorrer: uma página, um formulário e uma chamada interna entre os dois serviços.

## Aplicação

O visitante abre o frontend. O serviço lê `User-Agent`, `Accept-Language` e `Accept` do pedido HTTP e envia esses campos, junto com uma nota opcional, para `POST /v1/client-info` na API. A página mostra os dois lados: o que o frontend encaminhou e o que a API enxergou na chamada interna.

| Serviço | Cloud Run | Ingress |
| --- | --- | --- |
| Frontend | `run-svc-frontend` | `internal-and-cloud-load-balancing`. A URL `*.run.app` não abre na internet. O load balancer alcança o serviço sem checagem de IAM. |
| API | `run-svc-backend` | `internal`. Só aceita chamada com identity token da service account da aplicação. |

Os dois usam a conta `sa-sec-app` e o conector `vpcconnector` do projeto host `vpc-host-0123`, com saída apenas para IPs privados. A API fica com uma instância.

## Pipeline

Arquivo: `.github/workflows/pipeline.yaml`. Dispara no push da `main` ou manualmente.

1. Autenticação com a service account de CI.
2. Bandit em `api/`, `web/` e `zap/`.
3. Build das imagens `frontend`, `backend` e `zap` no Artifact Registry, deploy dos dois serviços e publicação do job.
4. Espera `TARGET_URL/health/ready` responder 200. Esse endereço é o load balancer, não a URL direta do Cloud Run.
5. Executa o job `zap-security-scan`. Alerta **FAIL** (alto) reprova. Aviso médio fica no relatório.
6. Se reprovar, devolve 100% do tráfego para a revisão anterior de cada serviço.
7. O relatório vai para `gs://gcs_zap/dast/<run>/` e para o artifact `zap-report`.

## Associação no LB

Depois do primeiro deploy do frontend:

1. Crie um NEG serverless regional em `us-central1` apontando para `run-svc-frontend`.
2. Crie o backend service e anexe esse NEG.
3. Aponte o URL map do `lb-external-uscentral` (projeto `vpc-host-0123`) para esse backend.
4. O DNS de `TARGET_URL` precisa cair nesse balanceador. O valor configurado é `http://sec-app-lab.souzatech.cloud`.

Enquanto o NEG não estiver no load balancer, o job de DAST espera o health check e, se não responder, reprova e faz rollback.

## Variáveis do repositório

| Variável | Uso |
| --- | --- |
| `PROJECT_ID` | Projeto de serviço, `sec-lab-510100`. |
| `REGION` | `us-central1`. |
| `WIF_SA` | Service account de CI. |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Provider do Workload Identity. |
| `APP_SA` | Service account de execução dos dois serviços. |
| `ARTIFACT_REPO` | Repositório Docker, `repo-sec-web-app`. |
| `TARGET_URL` | URL que o ZAP varre. |

A conta de CI precisa de `roles/iam.serviceAccountUser` em `APP_SA` e em `zap-job`, e de escrita no Artifact Registry.

## Local

```powershell
docker compose up --build
```

O frontend fica em [http://127.0.0.1:8080](http://127.0.0.1:8080) e a API em [http://127.0.0.1:8081](http://127.0.0.1:8081). No compose a chamada interna não usa identity token.
