# 01 — Visão geral do Openrouteservice

## O que é

O [openrouteservice](https://github.com/GIScience/openrouteservice) (ORS) é um serviço de roteamento open source (LGPL 3.0) escrito em Java 17 / Spring Boot, mantido pela HeiGIT (Universidade de Heidelberg). Usa um fork do GraphHopper como motor de grafos e consome mapas do OpenStreetMap (arquivos `.osm.pbf`).

Este repositório é um fork em `vertuam-jido-group/openrouteservice`. O deploy é feito com o **jar executável do Spring Boot** (`ors.jar`), compilado deste fork (ou baixado dos releases oficiais) e rodando direto na VPS via systemd — sem Docker.

## Endpoints disponíveis

Todos sob o prefixo `/ors/v2`:

| Endpoint | Função |
|---|---|
| `/directions/{perfil}` | Cálculo de rota entre pontos (GET ou POST) |
| `/isochrones/{perfil}` | Área alcançável a partir de um ponto (tempo/distância) |
| `/matrix/{perfil}` | Matriz de tempos/distâncias N×N |
| `/snap/{perfil}` | "Cola" coordenadas na malha viária mais próxima |
| `/export/{perfil}` | Exporta o grafo base |
| `/health` | `{"status":"ready"}` quando pronto — usar em monitoramento |
| `/status` | Informações detalhadas da instância |

Perfil habilitado no nosso setup: `driving-car`.

> **Atenção:** as coordenadas são sempre `longitude,latitude` (nessa ordem), padrão GeoJSON.

## Exemplo de chamada

```bash
curl -H "x-api-key: SUA-CHAVE" "https://rotas.jido.cab/ors/v2/directions/driving-car?start=-49.2733,-25.4284&end=-49.2306,-25.4950"
```

(Rota em Curitiba: start/end em `lon,lat`.)

## Dimensionamento de RAM

Regra de bolso oficial: **`XMX = tamanho-do-PBF × nº-de-perfis × 2`**

| Mapa | PBF | Heap p/ 1 perfil | VPS mínima |
|---|---|---|---|
| **Região Sul (nosso caso)** | ~700 MB | ~1.4 GB → usamos 2.5g | **4 GB** ✅ |
| Sudeste | ~700 MB | ~1.5 GB | 4–8 GB |
| Brasil inteiro | ~2 GB | ~8 GB | 16 GB |

Observações:

- O **pico de RAM é na construção do grafo** (primeira subida ou rebuild). Depois, o consumo cai.
- A Geofabrik **não oferece o Paraná isolado** — o menor recorte que o contém é a Região Sul (PR+SC+RS). Bônus: rotas interestaduais no Sul funcionam.
- Com 4 GB de RAM, **não habilitar um segundo perfil** (ex.: `driving-hgv`/caminhão) no mesmo container. Se precisar, subir a VPS para 8 GB.
- Elevação (relevo): o **padrão do ORS é ativada**, mas nossa config desativa (`elevation: false`) — o download dos tiles SRTM/CGIAR é instável e chega a derrubar o build do grafo (erro `cannot load srtm_... Expected 'GH' as file marker`, seguido de `Could not parse OSM file`). Para carro não faz falta.

## Limitações conhecidas

- **Sem autenticação nativa** — a proteção é feita no nginx (API key). Ver [03-nginx-https.md](03-nginx-https.md).
- Geocodificação (endereço → coordenada) **não faz parte** do ORS. Se precisar, é outro serviço (ex.: Pelias/Nominatim).
- Otimização de frota (tipo VRP/caixeiro-viajante) também não — para isso existe o [VROOM](https://github.com/VROOM-Project/vroom), que inclusive usa o ORS como backend.
