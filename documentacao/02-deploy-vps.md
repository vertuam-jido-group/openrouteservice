# 02 — Deploy na VPS (jar + systemd, sem Docker)

Pré-requisito: VPS Ubuntu/Debian com 4 GB RAM e acesso root/sudo.

## 1. Gerar o jar (na máquina de desenvolvimento)

No diretório do repositório (este fork), com Java 17 instalado:

```bash
./mvnw -DskipTests package -pl ors-api -am
```

O artefato final fica em **`ors-api/target/ors.jar`** (~150 MB, jar executável do Spring Boot com tudo embutido).

> Alternativa sem compilar: baixar o `ors.jar` pronto da página de [releases oficiais](https://github.com/GIScience/openrouteservice/releases) (asset `ors.jar`). Compilar do fork só é necessário se houver modificações próprias no código.

> **Não compile na VPS** — o build do Maven consome RAM e CPU que a VM de 4 GB não tem sobrando. Compile local e envie o jar.

## 2. Instalar o Java na VPS (17 ou superior)

O jar tem bytecode Java 17 e roda em qualquer JVM 17+. Na VPS atual já existe o **OpenJDK 25** — serve, não precisa instalar nada. Se fosse uma VM zerada:

```bash
sudo apt update && sudo apt install -y openjdk-17-jre-headless
```

> Com JVMs novas (21+) podem aparecer warnings de `sun.misc.Unsafe` / "restricted methods" no journal — são avisos das libs internas do GraphHopper, não são erros.

Verifique:

```bash
java -version
```

## 3. Criar swap de 2 GB (importante com 4 GB de RAM)

O build do grafo é o momento de pico de memória. O swap evita que o processo seja morto pelo kernel (OOM kill):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verifique com `free -h` (deve mostrar 2.0Gi de Swap).

## 4. Usuário e estrutura de diretórios

Criar um usuário de serviço dedicado (sem login) e o diretório da aplicação:

```bash
sudo useradd --system --home /opt/jido-rota --shell /usr/sbin/nologin ors
sudo mkdir -p /opt/jido-rota/files /opt/jido-rota/graphs /opt/jido-rota/logs /opt/jido-rota/elevation_cache
```

Layout final:

| Caminho | Conteúdo |
|---|---|
| `/opt/jido-rota/ors.jar` | A aplicação |
| `/opt/jido-rota/ors-config.yml` | Configuração |
| `/opt/jido-rota/files/` | O mapa `.osm.pbf` |
| `/opt/jido-rota/graphs/` | O grafo construído — **é o que vale backup** |
| `/opt/jido-rota/logs/` | Logs da aplicação |
| `/opt/jido-rota/elevation_cache/` | Cache de elevação (não usado no nosso setup) |

## 5. Baixar o mapa da Região Sul

```bash
sudo wget -P /opt/jido-rota/files https://download.geofabrik.de/south-america/brazil/sul-latest.osm.pbf
```

(~700 MB. A Geofabrik atualiza esse arquivo diariamente.)

## 6. Enviar o jar e a configuração

Da máquina de desenvolvimento:

```bash
scp ors-api/target/ors.jar documentacao/arquivos/ors-config.yml usuario@IP-DA-VPS:/tmp/
```

Na VPS:

```bash
sudo mv /tmp/ors.jar /tmp/ors-config.yml /opt/jido-rota/ && sudo chown -R ors:ors /opt/jido-rota
```

A config ([arquivos/ors-config.yml](arquivos/ors-config.yml)) já define:

- `server.port: 8082` e context-path `/ors` (casa com o nginx do doc 03)
- `source_file: /opt/jido-rota/files/sul-latest.osm.pbf`
- Grafo em `/opt/jido-rota/graphs`, logs em `/opt/jido-rota/logs`
- Perfil `driving-car` habilitado

O jar encontra a config pela variável `ORS_CONFIG_LOCATION` (definida na unidade systemd) — ordem de busca no código: [ORSEnvironmentPostProcessor.java](../ors-api/src/main/java/org/heigit/ors/api/ORSEnvironmentPostProcessor.java).

## 7. Serviço systemd

Copie [arquivos/ors.service](arquivos/ors.service) para `/etc/systemd/system/ors.service` e ative:

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now ors
```

Pontos-chave da unidade:

- `-Xms1g -Xmx2500m` — heap máximo de 2.5 GB, deixa ~1.5 GB para o SO
- `TimeoutStartSec=infinity` — o build do grafo na primeira subida demora; o systemd não pode matar o processo
- `Restart=on-failure` — sobe sozinho se cair
- Roda como usuário `ors`, sem privilégios

## 8. Primeira subida

Acompanhe o build do grafo (leva **~15–40 min** dependendo da CPU):

```bash
sudo journalctl -u ors -f
```

Espere pela mensagem indicando que o serviço está pronto (`ORS init complete` / profile `driving-car` ready).

## 9. Teste local (na própria VM)

Health check:

```bash
curl http://localhost:8082/ors/v2/health
```

Deve retornar `{"status":"ready"}`.

Rota de teste em Curitiba (formato `lon,lat`):

```bash
curl "http://localhost:8082/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```

Se retornou um GeoJSON com a rota, o backend está pronto. Siga para [03-nginx-https.md](03-nginx-https.md).

## Problemas comuns nesta etapa

| Sintoma | Causa provável | Solução |
|---|---|---|
| Processo morto durante o build (`Killed` / OOM no journal) | Falta de RAM | Confirmar swap ativo (`free -h`); reduzir `-Xmx` para `2200m` no ors.service; parar outros serviços durante o build |
| `health` retorna `not ready` por muito tempo | Grafo ainda em construção | Normal na primeira subida — acompanhar `journalctl -u ors -f` |
| Erro apontando o `source_file` | Caminho errado ou permissão | Conferir `/opt/jido-rota/files/sul-latest.osm.pbf` e `chown -R ors:ors /opt/jido-rota` |
| `Could not parse OSM file` precedido de `WARN CGIARProvider cannot load srtm_...` | Elevação ativada e download SRTM/CGIAR falhou (cache corrompido) | `sudo rm -f /opt/jido-rota/elevation_cache/*`; garantir `elevation: false` no bloco `build` do `ors-config.yml`; reiniciar |
| `Port ... already in use` | Outro serviço na mesma porta | `sudo ss -ltnp \| grep 8082` para ver o conflito; trocar a porta no `ors-config.yml` e no `proxy_pass` do nginx |
| Erros de caminho (`graphs`, `source_file` no lugar errado) | Config não encontrada (caiu no default) | Conferir `ORS_CONFIG_LOCATION` no ors.service e se `/opt/jido-rota/ors-config.yml` existe e é legível pelo usuário `ors` |
| `Permission denied` em graphs/logs | Dono errado após scp | `sudo chown -R ors:ors /opt/jido-rota` |
