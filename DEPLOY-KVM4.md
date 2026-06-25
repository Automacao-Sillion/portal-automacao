# Como hospedar o Portal de Automações no KVM4

## Ideia geral

O portal é **HTML estático** (index.html + setores). Ele não precisa de backend para
funcionar — só de um servidor web. Cada automação continua rodando **no seu próprio
container Docker/Streamlit**, exatamente como hoje. O portal apenas **embute** esses
apps via iframe.

```
Navegador
   │
   ▼
http://IP-DO-KVM4/            ← nginx servindo a pasta do portal (container "sillion-portal")
   │  (iframes)
   ├── http://IP-DO-KVM4:8501 ← container streamlit: Cartões
   ├── http://IP-DO-KVM4:8502 ← container streamlit: Baixas Vale
   ├── http://IP-DO-KVM4:8503 ← container streamlit: Faturar por OS
   └── ...                     (cada aba = um container já existente)
```

---

## Opção A — Simples (porta direta) — *a que você escolheu*

1. **Copie a pasta** do portal para o KVM4, por exemplo `/opt/sillion-portal`
   (precisa conter: `index.html`, `home.html`, `financeiro.html`, `fiscal.html`,
   `faturamento.html`, `departamento_pessoal.html`, `docker-compose.yml`).

2. **Suba o nginx do portal:**
   ```bash
   cd /opt/sillion-portal
   docker compose up -d
   ```

3. **Acesse:** `http://IP-DO-KVM4/`

4. **Aponte os iframes** para os seus containers. Em cada arquivo de setor, troque o
   endereço dos apps Streamlit (hoje apontam para `*.streamlit.app`) para os seus:
   ```
   http://cartoes.streamlit.app/?embed=true   →   http://IP-DO-KVM4:8501/?embed=true
   ```

> **Regra de ouro (mixed content):** portal e apps no **mesmo esquema**.
> Se o portal abre em `http://`, os apps também em `http://`. Funciona perfeitamente
> com IP:porta. Só misture com HTTPS se TUDO for HTTPS.

### Ajustes no Streamlit de cada container (para embutir bem)
No `config.toml` (ou flags ao subir o container):
```toml
[server]
enableCORS = false
enableXsrfProtection = false   # permite upload de arquivos dentro do iframe
headless = true
```
E mantenha o `?embed=true` na URL (o portal já usa) para o app vir sem o menu próprio.

---

## Opção B — Reverse proxy (recomendada para crescer)

Em vez de expor várias portas, o nginx serve o portal **e** redireciona caminhos para
cada container — tudo na **mesma origem** (uma porta só, e HTTPS fácil depois).

`nginx.conf` (exemplo):
```nginx
server {
    listen 80;
    server_name portal.sillion.com.br;

    # portal (arquivos estáticos)
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # cada automação vira um caminho → container
    location /caju/ {
        proxy_pass http://caju:8501/;     # "caju" = nome do serviço no docker network
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;     # necessário p/ websocket do Streamlit
        proxy_set_header Connection "upgrade";
    }
    # ... repetir para cartoes, baixas, etc.
}
```
Nesse modo, os iframes do portal apontam para `/caju/?embed=true` (sem porta, sem IP),
e cada Streamlit precisa subir com `--server.baseUrlPath=caju`.
Vantagem: um único ponto de entrada, e dá para colocar HTTPS (Let's Encrypt) só no nginx.

---

## Resumo

- **Hospedar o portal** = 1 container nginx servindo a pasta (`docker compose up -d`).
- **As automações** = seus containers atuais, sem mudança.
- **Conectar** = ajustar o `data-src` dos iframes para os endereços dos containers.
- `file://` **não** embute apps; servir por `http(s)://` (nginx) resolve.
