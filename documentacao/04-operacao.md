# 04 — Operação e manutenção

## Comandos do dia a dia

```bash
sudo systemctl status ors      # status do serviço
sudo journalctl -u ors -f      # acompanhar logs em tempo real
sudo systemctl restart ors     # reiniciar (NÃO reconstrói o grafo)
sudo systemctl stop ors        # parar
sudo systemctl start ors       # iniciar
```

Logs da aplicação também ficam em `/opt/jido-rota/logs/ors.log`.

## Atualizar o mapa (recomendado: mensal)

O OSM evolui (ruas novas, mudanças de sentido). O ORS reusa o grafo existente em `/opt/jido-rota/graphs` enquanto ele existir — para forçar a reconstrução com um PBF novo:

```bash
sudo wget -O /opt/jido-rota/files/sul-latest.osm.pbf https://download.geofabrik.de/south-america/brazil/sul-latest.osm.pbf
sudo chown ors:ors /opt/jido-rota/files/sul-latest.osm.pbf
sudo systemctl stop ors
sudo mv /opt/jido-rota/graphs /opt/jido-rota/graphs.old && sudo -u ors mkdir /opt/jido-rota/graphs
sudo systemctl start ors
```

O serviço fica **indisponível durante o rebuild** (~15–40 min). Quando `curl http://localhost:8082/ors/v2/health` voltar a `ready`, remova o grafo antigo:

```bash
sudo rm -rf /opt/jido-rota/graphs.old
```

> Manter o `graphs.old` até confirmar o `ready` é o rollback: se o rebuild falhar, basta parar o serviço, restaurar a pasta e iniciar de novo.

## Atualizar a versão do ORS

1. Conferir os [releases](https://github.com/GIScience/openrouteservice/releases) e o CHANGELOG
2. Gerar o novo `ors.jar` (build local `./mvnw -DskipTests package -pl ors-api -am`, ou baixar do release)
3. Enviar para a VPS e substituir:

```bash
sudo systemctl stop ors && sudo mv /tmp/ors.jar /opt/jido-rota/ors.jar && sudo chown ors:ors /opt/jido-rota/ors.jar && sudo systemctl start ors
```

4. Versões novas podem exigir rebuild do grafo — se o log reclamar de incompatibilidade de grafo, seguir o fluxo de atualização de mapa acima (trocar a pasta `graphs`).

## Backup

O que vale backup:

| Item | Caminho | Observação |
|---|---|---|
| Grafo construído | `/opt/jido-rota/graphs/` | Evita rebuild ao migrar de VM |
| Configuração | `/opt/jido-rota/ors-config.yml` | |
| Unidade systemd | `/etc/systemd/system/ors.service` | |
| Config do nginx | `/etc/nginx/sites-available/rotas.jido.cab` | Contém a API key |

O PBF em `files/` e o próprio `ors.jar` **não precisam** de backup (baixa/gera de novo).

Exemplo de backup do grafo:

```bash
sudo tar czf /root/ors-graphs-$(date +%Y%m%d).tar.gz -C /opt/jido-rota graphs
```

## Monitoramento

- **Uptime:** apontar um monitor (UptimeRobot, Better Stack, etc.) para `https://rotas.jido.cab/ors/v2/health` esperando `"ready"`. Esse endpoint não exige API key.
- **Detalhes da instância:** `GET /ors/v2/status` (atrás da API key).
- **Recursos da VM:** `htop` ou `systemctl status ors` mostram RAM/CPU do processo Java.
- **Falhas de inicialização:** `sudo journalctl -u ors --since "1 hour ago"`.

## Troubleshooting

| Sintoma | Diagnóstico | Ação |
|---|---|---|
| `502 Bad Gateway` no nginx | Serviço parado ou ainda subindo | `systemctl status ors` / `journalctl -u ors -f` |
| `401` com a chave certa | Header errado ou chave divergente | Conferir header `x-api-key` e o valor no server block |
| Processo morto (`Killed`, exit por OOM) | Falta de RAM | Conferir swap (`free -h`); reduzir `-Xmx`; não habilitar 2º perfil |
| `Port ... already in use` na subida | Conflito de porta com outro serviço | `sudo ss -ltnp \| grep 8082`; trocar a porta no `ors-config.yml` e no nginx |
| Caminhos errados (graphs fora de `/opt/jido-rota`) | Config não carregada | Conferir `ORS_CONFIG_LOCATION` no ors.service e permissões do `ors-config.yml` |
| Rota "não encontrada" perto da borda do mapa | Ponto fora da Região Sul | O grafo só cobre PR/SC/RS — trocar o recorte se precisar de mais |
| `Could not find point` | Coordenada longe de via roteável ou **lat/lon invertidos** | Lembrar: ordem é `lon,lat` |
| Respostas lentas em matriz grande | Matriz N×N cresce quadraticamente | Limitar tamanho das matrizes no cliente; avaliar upgrade da VM |

## Limites padrão relevantes

O ORS vem com limites de proteção (configuráveis no `ors-config.yml` em `ors.endpoints.*`):

- Directions: até 50 waypoints por rota
- Matrix: até 3.500 células (ex.: 59×59) por request
- Isochrones: até 5 locations, alcance máximo padrão de 50 km / 3600 s

Se algum limite apertar o uso de vocês, dá para subir no `ors-config.yml` (ex.: `ors.endpoints.matrix.maximum_routes: 10000`) — lembrando que limites maiores = mais RAM/CPU por request.
