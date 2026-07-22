# Documentação — Deploy do Openrouteservice (ORS)

Setup do serviço de rotas da Vertuam/Jido em `rotas.jido.cab`.

> **Status: ✅ EM PRODUÇÃO** (deploy concluído em 22/07/2026)
> `https://rotas.jido.cab/ors/v2/health` → `{"status":"ready"}`

## Cenário

| Item | Valor |
|---|---|
| VPS | `ladybug.vps-kinghost.net` — 4 GB RAM + 2 GB swap |
| Cobertura | Região Sul do Brasil (PR + SC + RS) — 2,24 mi nodes / 2,90 mi edges |
| Perfil habilitado | `driving-car` (carro) |
| Elevação | **Desativada** (`elevation: false` — ver histórico abaixo) |
| Domínio | `rotas.jido.cab` (DNS + proxy **Cloudflare**, nuvem laranja) |
| Proxy local | nginx + HTTPS (certbot/Let's Encrypt) + API key + rate limit |
| Porta interna | **8082** (a 8080 já estava ocupada na VPS) |
| Deploy | Jar do Spring Boot (`ors.jar` 83 MB) + systemd, sem Docker |
| Diretório | `/opt/jido-rota` |
| Serviço/usuário | systemd `ors`, usuário de sistema `ors` |
| Java na VPS | OpenJDK 25 (jar com bytecode 17 — compatível) |
| Build do jar | Feito no Mac com JDK 22 (`./mvnw -DskipTests package -pl ors-api -am`) |

## Ordem de leitura / execução

1. [01-visao-geral.md](01-visao-geral.md) — o que é o ORS, endpoints, dimensionamento de RAM
2. [02-deploy-vps.md](02-deploy-vps.md) — build do jar, preparação da VM, swap, systemd, primeira subida
3. [03-nginx-https.md](03-nginx-https.md) — nginx, certificado, API key, rate limit, **Cloudflare**, Swagger
4. [04-operacao.md](04-operacao.md) — atualização de mapa, backup, monitoramento, troubleshooting
5. [05-uso-api.md](05-uso-api.md) — **como os sistemas chamam a API** (exemplos, erros comuns, gestão da chave)

## Arquivos prontos para copiar

Pasta [deploy/](deploy/) — pacote completo usado no deploy (inclui o `ors.jar` compilado):

- [deploy/LEIA-ME.md](deploy/LEIA-ME.md) — checklist resumido da subida
- [deploy/ors.jar](deploy/ors.jar) + [deploy/ors.jar.sha256](deploy/ors.jar.sha256)
- [deploy/ors-config.yml](deploy/ors-config.yml) → `/opt/jido-rota/`
- [deploy/ors.service](deploy/ors.service) → `/etc/systemd/system/`
- [deploy/rotas.jido.cab.conf](deploy/rotas.jido.cab.conf) → `/etc/nginx/sites-available/` (⚠️ substituir o placeholder da API key)

(A pasta [arquivos/](arquivos/) tem as mesmas configs, sem o jar.)

## Resumo da arquitetura

```
Internet
   │  HTTPS (443)
   ▼
Cloudflare (proxy, nuvem laranja — SSL Full)
   │  HTTPS (443)
   ▼
nginx  ── TLS (certbot) + API key (x-api-key) + rate limit 10r/s
   │  http://127.0.0.1:8082
   ▼
ors.jar (systemd "ors", usuário "ors", -Xms1g -Xmx2500m)
   │  Spring Boot 3.5 / bytecode Java 17 / rodando em OpenJDK 25
   ▼
Grafo Região Sul 2D (persistido em /opt/jido-rota/graphs — build só na 1ª vez)
```

## Histórico do deploy — problemas reais e soluções

Registro do que aconteceu na subida de 22/07/2026, para consulta futura:

| # | Problema | Causa | Solução |
|---|---|---|---|
| 1 | Build do jar falhou no Mac (`cannot find symbol getService()...`) | JDK 25 (GraalVM) incompatível com Lombok 1.18.34 — getters não eram gerados | Compilar com JDK 22 do Homebrew (`JAVA_HOME=/opt/homebrew/opt/openjdk@22/...`) |
| 2 | `Port 8080 was already in use` na VPS | Outro serviço já ocupava a 8080 | ORS movido para a **8082** (config + `proxy_pass` do nginx) |
| 3 | `Could not parse OSM file` com `WARN CGIARProvider cannot load srtm_...` | Elevação vem **ativada por padrão** no ORS; o download dos tiles SRTM/CGIAR falhou e corrompeu o cache, abortando o build do grafo | `rm -f /opt/jido-rota/elevation_cache/*` + `elevation: false` no bloco `build` do `ors-config.yml` |
| 4 | nginx não subiu: `zero size shared memory zone "ors"` | `limit_req zone=ors` referenciado no site, mas a `limit_req_zone` não tinha sido adicionada ao `http {}` do `nginx.conf` | `sed -i '/^http {/a\	limit_req_zone $binary_remote_addr zone=ors:10m rate=10r/s;' /etc/nginx/nginx.conf` |
| 5 | `HTTP 526` (Invalid SSL certificate) via Cloudflare | Domínio atrás do proxy Cloudflare em SSL Full; a origem ainda não tinha certificado válido | `certbot --nginx -d rotas.jido.cab` na VPS |
| 6 | Swagger não abre no browser | Browser não envia `x-api-key` → 401 do nginx (proposital) | Túnel SSH: `ssh -L 8082:localhost:8082 root@VPS` → `http://localhost:8082/ors/swagger-ui/index.html` |

## Verificação rápida (a qualquer momento)

```bash
curl https://rotas.jido.cab/ors/v2/health
```

→ `{"status":"ready"}` = tudo ok. Sem chave, qualquer outro endpoint responde `401`; com o header `x-api-key` correto, responde normalmente (exemplos em [05-uso-api.md](05-uso-api.md)).
