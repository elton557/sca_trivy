# Estágio de Verificação de Vulnerabilidades com Trivy

Este projeto implementa um estágio de verificação de vulnerabilidades em imagens Docker usando o Trivy no pipeline CI/CD do GitLab. O estágio `trivy_scan` realiza uma análise de segurança automatizada, garantindo que vulnerabilidades críticas sejam identificadas antes da implantação.

## Funcionalidades Principais

- **Trivy**: Ferramenta de análise de vulnerabilidades em contêineres e imagens Docker.
- **Integração CI/CD no GitLab**: Verificação contínua de segurança durante o ciclo de desenvolvimento.
- **Relatórios Detalhados**: Geração de relatório em JSON e formato de tabela com contagem de vulnerabilidades.
- **Notificação por E-mail**: Envio automático do relatório de vulnerabilidades por e-mail, com detalhamento das contagens de vulnerabilidades.

## Requisitos

### Variáveis de Ambiente

Certifique-se de configurar as seguintes variáveis de ambiente no GitLab para o correto funcionamento do estágio `trivy_scan`.

#### Proxy

| Variável           | Descrição                              |
|--------------------|----------------------------------------|
| `PROXY_DOCKER`     | URL do proxy HTTP para o Docker.       |

#### Configurações do Trivy

| Variável            | Descrição                                                              |
|---------------------|------------------------------------------------------------------------|
| `SERVER_TRIVY`      | URL do servidor Trivy (cliente/servidor).                              |
| `NEXUS_URL_DEV`     | URL do repositório de imagens Docker.                                  |
| `CI_PROJECT_NAME`   | Nome do projeto, gerado automaticamente pelo GitLab.                   |
| `VERSION`           | Versão da imagem a ser escaneada.                                      |

#### Configurações SMTP (para envio de e-mail)

| Variável               | Descrição                                       |
|------------------------|-------------------------------------------------|
| `HOST_SMTP_ZAP`        | Endereço do servidor SMTP.                      |
| `USER_SMTP_ZAP`        | Nome de usuário para autenticação no SMTP.      |
| `PASSWORD_ZAP`         | Senha de autenticação no SMTP.                  |
| `REMETENTE_EMAIL_ZAP`  | E-mail do destinatário para o envio do relatório.|

## Explicação do Pipeline

O estágio `trivy_scan` executa as seguintes etapas no pipeline CI/CD:

1. **Configuração do Proxy e Dependências**: O Trivy utiliza variáveis para configurar o proxy e o cache, além de instalar pacotes `jq` e `msmtp` para análise e envio de e-mails.

2. **Execução do Trivy**:
   - **Formato de Tabela**: Realiza a análise de vulnerabilidades e salva o resultado em `trivy_table.txt` para visualização rápida.
   - **Formato JSON**: Gera um relatório JSON detalhado para permitir análise e integração posterior.

3. **Análise e Contagem de Vulnerabilidades**:
   - Extração e contagem das vulnerabilidades de diferentes níveis de severidade (MEDIUM, HIGH, CRITICAL, LOW) usando `jq`.
   - Exibição das contagens no log para depuração.

4. **Notificação de Limite de Vulnerabilidades**:
   - Se o número de vulnerabilidades CRITICAL, HIGH, MEDIUM ou LOW exceder o limite, a pipeline é interrompida para correção antes da implantação.

5. **Configuração de Envio de E-mail**:
   - Configura o cliente `msmtp` para enviar o relatório de vulnerabilidades por e-mail.

6. **Envio de Relatório por E-mail**:
   - O relatório de vulnerabilidades e as contagens são enviados automaticamente para o e-mail configurado.

## Exemplo de Uso

### Pipeline GitLab CI/CD - Exemplo de `trivy_scan`:

```yaml
trivy_scan:
  stage: trivy_scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - echo "Running Trivy scan..."
    - apk add --no-cache jq msmtp
    - export TRIVY_DISABLE_VEX_NOTICE=true
    - export TRIVY_CACHE_DIR="/root/.docker/db-trivy"
    - trivy image --cache-dir $TRIVY_CACHE_DIR --server $SERVER_TRIVY --format table $NEXUS_URL_DEV/dsis-1/${CI_PROJECT_NAME}:$VERSION --scanners vuln > trivy_table.txt
    - trivy image --cache-dir $TRIVY_CACHE_DIR --server $SERVER_TRIVY --format json $NEXUS_URL_DEV/dsis-1/${CI_PROJECT_NAME}:$VERSION --scanners vuln > trivy_report.json
    - |
      medium_count=$(jq '[.Results[]? | .Vulnerabilities[]? | select(.Severity == "MEDIUM")] | length' trivy_report.json)
      high_count=$(jq '[.Results[]? | .Vulnerabilities[]? | select(.Severity == "HIGH")] | length' trivy_report.json)
      critical_count=$(jq '[.Results[]? | .Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length' trivy_report.json)
      low_count=$(jq '[.Results[]? | .Vulnerabilities[]? | select(.Severity == "LOW")] | length' trivy_report.json)
      echo "MEDIUM: $medium_count, HIGH: $high_count, CRITICAL: $critical_count, LOW: $low_count"
      if [ "$critical_count" -ge 80 ]; then exit 1; fi
      if [ "$high_count" -ge 80 ]; then exit 1; fi
      if [ "$medium_count" -ge 80 ]; then exit 1; fi
      if [ "$low_count" -ge 100 ]; then exit 1; fi
    - |
      echo "account gitlab" > /etc/msmtprc
      echo "host $HOST_SMTP_ZAP" >> /etc/msmtprc
      echo "port 587" >> /etc/msmtprc
      echo "auth on" >> /etc/msmtprc
      echo "user $USER_SMTP_ZAP" >> /etc/msmtprc
      echo "password $PASSWORD_ZAP" >> /etc/msmtprc
      echo "tls on" >> /etc/msmtprc
      echo "tls_starttls on" >> /etc/msmtprc
      echo "tls_certcheck off" >> /etc/msmtprc
      echo "from $USER_SMTP_ZAP" >> /etc/msmtprc
      echo "logfile /var/log/msmtp.log" >> /etc/msmtprc
      echo "account default : gitlab" >> /etc/msmtprc
    - chmod 600 /etc/msmtprc
    - |
      {
        current_time=$(date "+%Y-%m-%d")
        echo "Subject: Relatório de Vulnerabilidades Trivy - $current_time"
        echo "To: $REMETENTE_EMAIL_ZAP"
        echo "Content-Type: text/plain; charset=\"utf-8\""
        echo ""
        echo "Relatório gerado em $current_time."
        cat trivy_table.txt
        echo "MEDIUM: $medium_count, HIGH: $high_count, CRITICAL: $critical_count"
        echo "Revise as vulnerabilidades."
      } | msmtp -a gitlab -t
  tags:
    - docker
  allow_failure: false
