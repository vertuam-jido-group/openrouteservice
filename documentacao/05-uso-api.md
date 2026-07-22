# 05 — Uso da API (integração dos sistemas)

A API está pública em `https://rotas.jido.cab`, protegida por API key. Toda chamada (exceto `/health`) deve enviar o header:

```
x-api-key: <A-CHAVE-CONFIGURADA-NO-NGINX>
```

## A API key

- A chave é comparada **literalmente** pelo nginx com o valor gravado em `/etc/nginx/sites-available/rotas.jido.cab` — não é "qualquer valor", tem que ser exatamente a string configurada.
- Ver a chave atual (na VPS):

```bash
sudo grep -n "http_x_api_key" /etc/nginx/sites-available/rotas.jido.cab
```

- Trocar a chave (rotação):

```bash
openssl rand -hex 32   # gera a nova
sudo sed -i 's/CHAVE-ANTIGA/CHAVE-NOVA/' /etc/nginx/sites-available/rotas.jido.cab
sudo nginx -t && sudo systemctl reload nginx
```

(Sem downtime — o `reload` é instantâneo. Lembre de atualizar a chave nos sistemas consumidores.)

- **Nunca colocar a chave em frontend público** (site/app): qualquer um vê no DevTools do browser. Frontend chama um backend da Vertuam; o backend chama o `rotas.jido.cab` com a chave.
- Guardar a chave em secrets/variável de ambiente dos sistemas, nunca hardcoded em repositório.

## Regra de ouro das coordenadas

**Sempre `[longitude, latitude]`** (padrão GeoJSON) — nessa ordem. Inverter é o erro nº 1 de integração e causa `Could not find point`. Curitiba, por exemplo: `-49.2733, -25.4284` (lon, lat).

## Endpoints e exemplos

### Rota simples (GET)

```bash
curl -H "x-api-key: SUA-CHAVE" "https://rotas.jido.cab/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```

### Rota com múltiplos pontos / opções (POST)

```javascript
const res = await fetch("https://rotas.jido.cab/ors/v2/directions/driving-car/geojson", {
  method: "POST",
  headers: { "x-api-key": process.env.ORS_API_KEY, "Content-Type": "application/json" },
  body: JSON.stringify({
    coordinates: [[-49.2733, -25.4284], [-49.2306, -25.4950]],
    instructions: true,
    language: "pt"          // instrucoes de navegacao em portugues
  })
});
const rota = await res.json();
```

### Matriz de distâncias/tempos (roteirização, entregas)

```javascript
const res = await fetch("https://rotas.jido.cab/ors/v2/matrix/driving-car", {
  method: "POST",
  headers: { "x-api-key": process.env.ORS_API_KEY, "Content-Type": "application/json" },
  body: JSON.stringify({
    locations: [[-49.2733, -25.4284], [-49.2306, -25.4950], [-49.3044, -25.4950]],
    metrics: ["distance", "duration"]
  })
});
```

### Isócronas (área alcançável em X minutos)

```javascript
const res = await fetch("https://rotas.jido.cab/ors/v2/isochrones/driving-car", {
  method: "POST",
  headers: { "x-api-key": process.env.ORS_API_KEY, "Content-Type": "application/json" },
  body: JSON.stringify({
    locations: [[-49.2733, -25.4284]],
    range: [600, 1200]      // 10 e 20 minutos, em segundos
  })
});
```

### Snap (colar coordenada na via mais próxima)

```bash
curl -H "x-api-key: SUA-CHAVE" -H "Content-Type: application/json" -X POST "https://rotas.jido.cab/ors/v2/snap/driving-car" -d '{"locations":[[-49.2733,-25.4284]],"radius":300}'
```

### Health (monitoramento — único endpoint SEM chave)

```bash
curl https://rotas.jido.cab/ors/v2/health
```

→ `{"status":"ready"}`

## Swagger / documentação interativa

O Swagger UI **não abre pelo browser em `rotas.jido.cab`** — o browser não envia o header `x-api-key`, então cai no 401 do nginx (comportamento proposital). Opções:

1. **Túnel SSH** (recomendado — nada exposto):

```bash
ssh -L 8082:localhost:8082 root@IP-DA-VPS
```

Com o túnel aberto: `http://localhost:8082/ors/swagger-ui/index.html`

2. **Documentação oficial** (mesmos endpoints): https://giscience.github.io/openrouteservice/api-reference/
3. Desativar o Swagger em produção (opcional, boa prática) — adicionar no `/opt/jido-rota/ors-config.yml`:

```yaml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

e `sudo systemctl restart ors`.

## Limites e cobertura

- **Cobertura do mapa:** Região Sul (PR, SC, RS). Pontos fora dessa área → erro de rota. Bounds do grafo: lon -57.70 a -48.02, lat -33.76 a -22.40.
- **Limites padrão** (configuráveis em `ors.endpoints.*` no `ors-config.yml`): 50 waypoints por rota; matriz de até 3.500 células; isócronas com até 5 locations e alcance de 50 km / 3600 s.
- **Rate limit no nginx:** 10 req/s com burst de 20. Observação: com o proxy do Cloudflare ativo, o nginx vê IPs do Cloudflare — o limite vale "por IP do CF", menos preciso por cliente real (ver nota em [03-nginx-https.md](03-nginx-https.md)).

## Erros comuns de integração

| Resposta | Causa | Correção |
|---|---|---|
| `401 {"error": "unauthorized"}` | Header `x-api-key` ausente ou chave errada | Conferir header e valor exato da chave |
| `Could not find point` / ponto não roteável | Lat/lon invertidos, ou ponto fora da Região Sul | Ordem é `lon,lat`; conferir cobertura |
| `404` em `/directions` sem perfil | URL sem o perfil | Usar `/ors/v2/directions/driving-car` |
| `406`/`400` em POST | `Content-Type: application/json` faltando ou body malformado | Conferir header e JSON |
| `503`/`502` | ORS reiniciando ou rebuild de grafo em andamento | Conferir `https://rotas.jido.cab/ors/v2/health` |
