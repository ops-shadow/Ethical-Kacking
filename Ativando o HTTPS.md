# 📌 Habilitando conteúdo HTTPS no Apache2 do Kali Linux

você precisa **ativar o módulo SSL** e **configurar um certificado digital** (pode ser autoassinado ou de uma autoridade como o Let’s Encrypt).

## 1. Ativar módulo SSL

```
sudo a2enmod ssl
```

## 2. Criar ou obter certificado

* **Certificado autoassinado** (para testes):

```
sudo mkdir /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
 -keyout /etc/apache2/ssl/apache.key \
 -out /etc/apache2/ssl/apache.crt
```

Durante o comando, ele vai pedir informações (país, organização, etc.).

* **Certificado real** (produção):

  * Pode usar **Let’s Encrypt** com `certbot`.

## 3. Criar Virtual Host HTTPS

Crie o arquivo:

```
sudo nano /etc/apache2/sites-available/ssl.conf
```

Conteúdo básico:

apache
```
<VirtualHost *:443>
    ServerAdmin admin@meusite.com
    DocumentRoot /var/www/html
    ServerName meusite.com

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/apache.crt
    SSLCertificateKeyFile /etc/apache2/ssl/apache.key

    <Directory /var/www/html>
        AllowOverride All
    </Directory>
</VirtualHost>
```

## 4. Ativar site SSL e reiniciar Apache

```
sudo a2ensite ssl.conf
sudo systemctl restart apache2
```

## 5. Testar no navegador

Acesse:

```
https://localhost
```

ou o IP do seu Kali.
Se for certificado autoassinado, o navegador vai mostrar um aviso de segurança (normal para testes).

---
# Passo-a-passo direto, testado em Debian/Kali, para ativar HTTPS no Apache2 usando Certbot/Let’s Encrypt
*(certificado gratuito, público e aceito pelos navegadores)*
---
**Dois caminhos:**

* **Caminho A (mais simples)**: validação **HTTP-01** – exige o domínio apontando para o seu servidor e **porta 80 aberta**.
* **Caminho B**: validação **DNS-01** – serve quando você **não pode abrir a porta 80** (ou quer **wildcard** `*.seu-dominio.com`).

> Importante: a Let’s Encrypt só emite certificados para **domínios públicos que você controla**. Para HTTP-01 a autoridade valida **via porta 80**; se ela estiver fechada, use DNS-01. ([letsencrypt.org][1])

---

## Pré-requisitos

1. **Domínio** apontando para o IP público do seu Kali (registros A/AAAA).
2. **Apache2** instalado e ativo:

   ```bash
   sudo apt update
   sudo apt install -y apache2
   sudo systemctl enable --now apache2
   ```
3. **Portas abertas** no roteador/firewall:

   * HTTP (80) e HTTPS (443) para o **Caminho A**.
   * Para **Caminho B (DNS-01)** a 80 pode ficar fechada. ([letsencrypt.org][1])

---

## 1) Prepare o VirtualHost no Apache

Crie uma raiz do site e um vhost simples em **:80** (o Certbot parte deste host para automatizar o :443):

```bash
sudo mkdir -p /var/www/seu-dominio.com/public
echo '<h1>OK</h1>' | sudo tee /var/www/seu-dominio.com/public/index.html

sudo tee /etc/apache2/sites-available/seu-dominio.com.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName seu-dominio.com
    ServerAlias www.seu-dominio.com
    DocumentRoot /var/www/seu-dominio.com/public

    <Directory /var/www/seu-dominio.com/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/seu-dominio_error.log
    CustomLog ${APACHE_LOG_DIR}/seu-dominio_access.log combined
</VirtualHost>
EOF

sudo a2ensite seu-dominio.com.conf
sudo a2dissite 000-default.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

> Ter um site HTTP funcional na porta 80 simplifica o `--apache` do Certbot. ([Certbot][2])

---

# 2) Instale o Certbot (plugin do Apache)

No Kali (base Debian), use os pacotes oficiais:

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-apache
certbot --version
```

> O pacote `python3-certbot-apache` é o plugin que integra o Certbot ao Apache. ([Debian Packages][3])

> Alternativa oficial é instalar por **snap** ou **pip**, mas o fluxo de uso continua `certbot --apache`. ([Certbot][4])

---

## 3) CAMINHO A — Emitir e instalar com HTTP-01 (recomendado)

Com as portas **80/443** acessíveis da Internet, rode:

```bash
sudo certbot --apache -d seu-dominio.com -d www.seu-dominio.com
```

* Escolha a opção que **força redirecionamento para HTTPS** quando o Certbot perguntar (ou rode não-interativo):

  ```bash
  sudo certbot --apache -d seu-dominio.com -d www.seu-dominio.com \
    --redirect --agree-tos -m seuemail@exemplo.com --no-eff-email -n
  ```

Isso:

* Prova o controle do domínio via **HTTP-01** (precisa da **porta 80**)
* Gera os arquivos em `/etc/letsencrypt/live/seu-dominio.com/`
* Cria/edita o vhost **SSL** (arquivo `seu-dominio.com-le-ssl.conf`) e recarrega o Apache. ([eff-certbot.readthedocs.io][5], [Certbot][2])

Teste: acesse `https://seu-dominio.com`.

---

### Renovação automática

Os certificados valem **90 dias**. O pacote do Certbot configura **renovação automática** via systemd timer. Teste assim:

```bash
sudo certbot renew --dry-run
```

Se o teste passar, sua renovação está OK. ([Reintech][6])

---

## 4) (Opcional) Endureça o TLS

O Certbot já habilita um conjunto seguro por padrão. Você pode reforçar cabeçalhos em seu arquivo `*-le-ssl.conf`, por exemplo:

```apache
# dentro do <VirtualHost *:443>
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
```

Depois:

```bash
sudo a2enmod headers
sudo systemctl reload apache2
```

> Ajustes de protocolo/ciphers e HSTS são boas práticas após instalar o certificado. ([Reintech][6])

---

## 5) CAMINHO B — Sem porta 80? Use **DNS-01** (inclui wildcard)

Se você não consegue abrir a **porta 80** (NAT, ISP bloqueia, etc.) ou quer **`*.seu-dominio.com`**, faça a validação por **DNS-01**. Há dois jeitos:

### B1) Usar plugin DNS do seu provedor (recomendado para auto-renovação)

Exemplo com Cloudflare (troque pelo seu provedor):

```bash
sudo apt install -y python3-certbot-dns-cloudflare
mkdir -p ~/.secrets/certbot
chmod 700 ~/.secrets/certbot
nano ~/.secrets/certbot/cloudflare.ini   # adicione: dns_cloudflare_api_token = <token>
chmod 600 ~/.secrets/certbot/cloudflare.ini

sudo certbot --authenticator dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini \
  -d seu-dominio.com -d '*.seu-dominio.com' --agree-tos -m seuemail@exemplo.com --no-eff-email --deploy-hook "systemctl reload apache2"
```

Depois crie (ou edite) o seu vhost `*-le-ssl.conf` para atender os subdomínios desejados.

> **Por quê DNS-01?** Porque o desafio é feito por um **registro TXT no DNS**, dispensando a porta 80 e permitindo **wildcards**. ([letsencrypt.org][7])

### B2) Modo manual (sem plugin)

Funciona, mas **não renova sozinho** (você terá que repetir o processo a cada 60–90 dias):

```bash
sudo certbot -d seu-dominio.com -d '*.seu-dominio.com' \
  --manual --preferred-challenges dns --agree-tos -m seuemail@exemplo.com
```

O Certbot mostrará o **TXT** que você precisa criar no DNS. Após propagar, ele emite o certificado. ([eff-certbot.readthedocs.io][5])
[6]: https://reintech.io/blog/secure-apache-lets-encrypt-debian-12?utm_source=chatgpt.com "Secure Apache with Let's Encrypt on Debian 12 - Reintech"
[7]: https://letsencrypt.org/docs/challenge-types/?utm_source=chatgpt.com "Challenge Types"

