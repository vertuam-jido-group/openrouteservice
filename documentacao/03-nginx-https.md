# 03 — nginx, HTTPS e API key

Pré-requisito: o serviço ORS rodando e respondendo em `http://127.0.0.1:8082` ([02-deploy-vps.md](02-deploy-vps.md)).

## 1. DNS

No painel DNS do domínio `jido.cab`, criar o registro:

```
Tipo: A
Nome: rotas
Valor: <IP da VPS>
TTL: 300 (ou padrão)
```

Confirme a propagação antes de seguir:

```bash
dig +short rotas.jido.cab
```

Deve retornar o IP da VPS.

## 2. Server block do nginx

Copie [arquivos/rotas.jido.cab.conf](arquivos/rotas.jido.cab.conf) para a VM em `/etc/nginx/sites-available/rotas.jido.cab`.

**Antes de ativar**, gere a API key e substitua o placeholder no arquivo:

```bash
openssl rand -hex 32
```

Edite o arquivo e troque `TROCAR-PELA-CHAVE-GERADA` pela chave gerada. Guarde essa chave — é ela que os sistemas da Vertuam vão enviar no header `x-api-key`.

Ative o site:

```bash
sudo ln -s /etc/nginx/sites-available/rotas.jido.cab /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## 3. Rate limit (zona no nginx.conf)

O server block usa uma zona de rate limit que precisa ser declarada no bloco `http {}` do `/etc/nginx/nginx.conf`. Adicione dentro de `http { ... }`:

```nginx
limit_req_zone $binary_remote_addr zone=ors:10m rate=10r/s;
```

Depois:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 4. HTTPS com certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d rotas.jido.cab
```

O certbot edita o server block sozinho: adiciona o bloco 443 com o certificado e o redirect 80 → 443. A renovação é automática (timer do systemd); confira com:

```bash
sudo certbot renew --dry-run
```

## 5. Firewall (se usar ufw)

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

A porta 8082 **não** deve ser aberta — o ORS só escuta em localhost.

## 6. Verificação final

Health (sem chave — liberado para monitoramento):

```bash
curl https://rotas.jido.cab/ors/v2/health
```

Rota **sem** chave — deve retornar `401`:

```bash
curl -i "https://rotas.jido.cab/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```

Rota **com** chave — deve retornar o GeoJSON:

```bash
curl -H "x-api-key: SUA-CHAVE" "https://rotas.jido.cab/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```

## Cloudflare (o domínio jido.cab usa proxy do Cloudflare)

O `rotas.jido.cab` está atrás do proxy do Cloudflare (nuvem laranja). Consequências práticas:

- **Erro 526 (Invalid SSL certificate):** acontece quando o Cloudflare (modo SSL "Full strict") conecta na VPS e não encontra certificado válido para o hostname. Foi o que ocorreu no primeiro acesso — resolvido rodando o `certbot --nginx -d rotas.jido.cab` na origem. Se voltar a acontecer (ex.: certificado expirado sem renovar), conferir `sudo certbot certificates` e `sudo certbot renew`.
- **IP real do cliente:** o nginx enxerga os IPs do Cloudflare, não os dos clientes. Efeitos: o rate limit (10 r/s) vale "por IP do Cloudflare" (menos preciso) e um `allow/deny` por IP no nginx **não funciona** sem antes configurar `set_real_ip_from` + `real_ip_header CF-Connecting-IP`. A API key não é afetada.
- **Alternativa:** desativar o proxy (nuvem cinza / DNS only) deixa a conexão direta na VPS — mais simples, proteção fica toda no nginx. Para uma API consumida só por sistemas próprios, é suficiente.

## Swagger / documentação interativa

O Swagger não abre pelo browser público — o browser não envia `x-api-key` e recebe 401 (proposital). Acesso administrativo via túnel SSH:

```bash
ssh -L 8082:localhost:8082 root@IP-DA-VPS
```

→ `http://localhost:8082/ors/swagger-ui/index.html`. Detalhes e alternativas em [05-uso-api.md](05-uso-api.md).

## Gestão da API key

- A chave é o valor literal no `if ($http_x_api_key != "...")` do server block — ver com `sudo grep http_x_api_key /etc/nginx/sites-available/rotas.jido.cab`.
- **Nunca deixar o placeholder `TROCAR-PELA-CHAVE-GERADA`** — ele está público nesta documentação e funcionaria como chave.
- Rotação de chave (sem downtime): gerar nova com `openssl rand -hex 32`, `sed` no arquivo, `nginx -t && systemctl reload nginx`, atualizar os consumidores.

## Problemas encontrados no deploy real (e soluções)

| Sintoma | Causa | Solução |
|---|---|---|
| nginx não sobe: `zero size shared memory zone "ors"` | O site referencia `limit_req zone=ors`, mas a diretiva `limit_req_zone` não foi adicionada ao `http {}` do `nginx.conf` | `sudo sed -i '/^http {/a\	limit_req_zone $binary_remote_addr zone=ors:10m rate=10r/s;' /etc/nginx/nginx.conf` e `nginx -t && systemctl start nginx` |
| `HTTP 526` vindo do Cloudflare | Sem certificado válido na origem para o hostname | `sudo certbot --nginx -d rotas.jido.cab` |
| `[warn] protocol options redefined` no `nginx -t` | Redefinição de opções TLS entre server blocks de outros sites da VPS (pré-existente) | Inofensivo; não impede o start |
| Swagger dá 401 no browser | Browser não envia `x-api-key` | Túnel SSH (ver acima) |

## Notas de segurança

- **A API key no nginx protege contra uso anônimo**, mas ela viaja em cada request — use sempre HTTPS (já garantido pelo certbot).
- **Nunca colocar essa chave em frontend público** (site/app): qualquer um vê no DevTools. Se um dia um frontend com mapa precisar consumir o serviço, o caminho certo é o frontend chamar um backend da Vertuam, e esse backend chamar o `rotas.jido.cab` com a chave.
- Alternativa mais restritiva: se só servidores conhecidos consomem, trocar a API key por allowlist de IP no `location /`:

  ```nginx
  allow <IP-do-servidor-consumidor>;
  deny all;
  ```
