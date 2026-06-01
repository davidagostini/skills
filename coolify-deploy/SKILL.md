---
name: coolify-deploy
description: >-
  Skill para fazer deploy no Coolify (PaaS self-hosted). Use sempre que precisar
  criar ou configurar um docker-compose para o Coolify, definir variáveis de ambiente,
  configurar redes compartilhadas entre projetos, expor serviços via domínio, ou
  entender as diferenças do Coolify em relação ao docker compose local.
  Ativa em contextos como: "subir no Coolify", "deploy Coolify", "coolify-compose",
  "configurar projeto Coolify", "domínio no Coolify", "variáveis no Coolify",
  "rede compartilhada Coolify", "self-hosted PaaS", ou qualquer menção a Coolify.
  Cobre: compose auto-contido com configs embutidas, redes entre projetos, HTTPS
  automático via Traefik, variáveis obrigatórias e opcionais, postgres_exporter,
  node_exporter e stacks de observabilidade multi-app.
---

# Coolify Deploy — Skill

**Coolify** é um PaaS self-hosted (alternativa ao Heroku/Railway). Roda no seu VPS e
usa Docker Compose por baixo — mas tem algumas peculiaridades importantes.

---

## 1. Diferenças Coolify vs docker compose local

| Aspecto | Local | Coolify |
|---|---|---|
| Configs embutidas (`configs:`) | Bind-mounts de arquivos | **Usar `configs:` com `content:`** — sem arquivos locais |
| Redes | Qualquer nome | Coolify gerencia a rede do projeto automaticamente |
| Portas | Expostas no host | Coolify faz proxy reverso — só expor a porta que o Traefik usa |
| Variáveis de ambiente | `.env` | Configuradas na UI do Coolify |
| Volumes | Locais | Gerenciados pelo Coolify |
| `container_name` | Opcional | Evitar — Coolify atribui nomes próprios |

---

## 2. Compose auto-contido (sem bind-mounts)

A forma correta para o Coolify é embutir configs no próprio compose usando `configs:`:

```yaml
configs:
  minha_config:
    content: |
      # conteúdo do arquivo aqui
      chave: valor
      # variáveis do Coolify são substituídas aqui:
      endpoint: ${MINHA_VAR:-valor-default}

services:
  meu-servico:
    image: minha-imagem:latest
    restart: unless-stopped
    configs:
      - source: minha_config
        target: /etc/app/config.yml
    environment:
      VARIAVEL: ${VARIAVEL:?VARIAVEL obrigatorio no Coolify}
```

**Vantagem:** o arquivo é 100% portável — nenhum arquivo externo precisa existir no servidor.

---

## 3. Variáveis de ambiente

```yaml
environment:
  # Obrigatória — falha se não definida (útil para forçar configuração)
  SENHA: ${SENHA:?Defina SENHA no painel do Coolify}

  # Opcional com default
  PORTA: ${PORTA:-8080}

  # Booleano
  DEBUG: ${DEBUG:-false}
```

No painel do Coolify: **Projeto → Environment Variables** — definir cada variável.

---

## 4. Redes compartilhadas entre projetos

Para que serviços em **projetos diferentes** do Coolify se comuniquem (ex: app + observabilidade):

```yaml
# No projeto A (cria a rede)
networks:
  minha-rede:
    name: minha-rede      # nome fixo — o Coolify reutiliza se já existir
    driver: bridge

# No projeto B (usa a mesma rede)
networks:
  minha-rede:
    name: minha-rede      # mesmo nome = mesma rede física
    driver: bridge
```

Serviços se comunicam pelo nome do serviço (não pelo container_name).

---

## 5. Expor serviço via domínio (HTTPS automático)

O Coolify usa **Traefik** como proxy reverso com HTTPS via Let's Encrypt.

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"    # Coolify usa esta porta para rotear pelo domínio
    environment:
      GF_SERVER_ROOT_URL: https://${GRAFANA_DOMAIN:?Defina GRAFANA_DOMAIN}
      GF_SERVER_DOMAIN: ${GRAFANA_DOMAIN}
```

No Coolify UI: **Serviço → Domain** — adicionar `grafana.seudominio.com`.
HTTPS é configurado automaticamente.

---

## 6. Healthchecks (importante para depends_on)

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s   # tempo para o app iniciar antes de checar

  worker:
    depends_on:
      api:
        condition: service_healthy  # aguarda api estar healthy
```

---

## 7. Padrão completo — stack de observabilidade no Coolify

```yaml
# deploy/observability/coolify-compose.yml
networks:
  obs-network:
    name: obs-network
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:

configs:
  prometheus_config:
    content: |
      global:
        scrape_interval: 15s
      scrape_configs:
        - job_name: "docker-auto"
          docker_sd_configs:
            - host: unix:///var/run/docker.sock
          relabel_configs:
            - source_labels: [__meta_docker_container_label_prometheus_scrape]
              action: keep
              regex: "true"

  grafana_datasources:
    content: |
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus:9090
          isDefault: true

services:
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=${RETENTION_DAYS:-30}d
    configs:
      - source: prometheus_config
        target: /etc/prometheus/prometheus.yml
    volumes:
      - prometheus_data:/prometheus
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [obs-network]

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:?GRAFANA_PASSWORD obrigatorio}
      GF_SERVER_ROOT_URL: https://${GRAFANA_DOMAIN:?GRAFANA_DOMAIN obrigatorio}
    configs:
      - source: grafana_datasources
        target: /etc/grafana/provisioning/datasources/datasources.yml
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks: [obs-network]
    depends_on: [prometheus]
```

---

## 8. Conectar app ao stack de observabilidade

No `docker-compose.yml` do app:

```yaml
services:
  api:
    labels:
      prometheus.scrape: "true"
      prometheus.port: "8080"
      prometheus.path: "/metrics"
    networks:
      - app-network
      - obs-network        # mesma rede que o Prometheus

networks:
  app-network:
    driver: bridge
  obs-network:
    name: obs-network      # mesmo nome = mesma rede do stack de observabilidade
    driver: bridge
```

---

## 9. Checklist antes de subir no Coolify

- [ ] Compose usa `configs:` com `content:` (sem bind-mounts de arquivos locais)?
- [ ] Variáveis obrigatórias têm `:?` para forçar configuração?
- [ ] Variáveis opcionais têm `:-default` para não quebrar sem valor?
- [ ] Sem `container_name` hardcoded?
- [ ] Healthchecks definidos nos serviços críticos?
- [ ] Redes com `name:` fixo para compartilhamento entre projetos?
- [ ] Domínio configurado na UI do Coolify para serviços expostos?
- [ ] Secrets (senhas, chaves) como variáveis de ambiente (não hardcoded)?
