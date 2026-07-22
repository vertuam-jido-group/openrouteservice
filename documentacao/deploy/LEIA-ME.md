# Pacote de deploy — rotas.jido.cab

Conteúdo desta pasta (tudo pronto para subir):

| Arquivo | Destino na VPS |
|---|---|
| `ors.jar` | `/opt/jido-rota/ors.jar` |
| `ors-config.yml` | `/opt/jido-rota/ors-config.yml` |
| `ors.service` | `/etc/systemd/system/ors.service` |
| `rotas.jido.cab.conf` | `/etc/nginx/sites-available/rotas.jido.cab` ⚠️ trocar `TROCAR-PELA-CHAVE-GERADA` antes |

## Passo a passo resumido

Documentação completa: [../02-deploy-vps.md](../02-deploy-vps.md) e [../03-nginx-https.md](../03-nginx-https.md).

### 1. Enviar os arquivos para a VPS

```bash
scp ors.jar ors-config.yml ors.service rotas.jido.cab.conf usuario@IP-DA-VPS:/tmp/
```

### 2. Na VPS — preparação (uma vez só)

```bash
sudo apt update && sudo apt install -y openjdk-17-jre-headless
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo useradd --system --home /opt/jido-rota --shell /usr/sbin/nologin ors
sudo mkdir -p /opt/jido-rota/files /opt/jido-rota/graphs /opt/jido-rota/logs /opt/jido-rota/elevation_cache
sudo wget -P /opt/jido-rota/files https://download.geofabrik.de/south-america/brazil/sul-latest.osm.pbf
```

### 3. Na VPS — instalar os arquivos

```bash
sudo mv /tmp/ors.jar /tmp/ors-config.yml /opt/jido-rota/
sudo chown -R ors:ors /opt/jido-rota
sudo mv /tmp/ors.service /etc/systemd/system/ors.service
sudo mv /tmp/rotas.jido.cab.conf /etc/nginx/sites-available/rotas.jido.cab
```

### 4. Subir o serviço

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now ors
sudo journalctl -u ors -f   # aguardar o build do grafo (~15–40 min)
```

Testar: `curl http://localhost:8082/ors/v2/health` → `{"status":"ready"}`

### 5. nginx + HTTPS

```bash
openssl rand -hex 32   # gerar a API key e colocar no rotas.jido.cab.conf
sudo nano /etc/nginx/sites-available/rotas.jido.cab   # substituir TROCAR-PELA-CHAVE-GERADA
# Declarar a zona de rate limit DENTRO do bloco http{} do nginx.conf (sem isso: "zero size shared memory zone"):
sudo sed -i '/^http {/a\	limit_req_zone $binary_remote_addr zone=ors:10m rate=10r/s;' /etc/nginx/nginx.conf
sudo ln -s /etc/nginx/sites-available/rotas.jido.cab /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d rotas.jido.cab
```

### 6. Verificação final

```bash
curl https://rotas.jido.cab/ors/v2/health
curl -H "x-api-key: SUA-CHAVE" "https://rotas.jido.cab/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```
