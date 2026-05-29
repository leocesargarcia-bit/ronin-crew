# Ronin Crew — Roteiro de Deploy
**Versão:** 2026-05  
**Stack:** HTML/CSS/JS estático + PrestaShop (e-commerce) + VPS Hostinger  
**Repositório alvo:** `github.com/[seu-user]/ronin-crew`

---

## FASE 0 — Pré-requisitos (fazer antes de tudo)

- [ ] Git instalado localmente → https://git-scm.com/download/win
- [ ] Conta GitHub criada → https://github.com
- [ ] Acesso ao painel Hostinger → https://hpanel.hostinger.com
- [ ] Dados de acesso SSH da VPS (IP, usuário root, senha ou chave SSH)
- [ ] PrestaShop baixado → https://www.prestashop.com/en/download (versão 8.x)
- [ ] Domínio configurado no Hostinger apontando para a VPS (ou Hostinger Shared)

---

## FASE 1 — GitHub: versionar o projeto

### 1.1 Criar repositório no GitHub
1. Acesse github.com → **New repository**
2. Nome: `ronin-crew-site`
3. Visibilidade: **Private** (até o launch)
4. Não inicializar com README (vamos enviar o código local)

### 1.2 Inicializar Git na pasta do projeto
Abra o terminal na pasta `D:\ronin\Claude\Site Ronin\` e execute:

```bash
git init
git add .
git commit -m "feat: Ronin Crew v2 — Tactical Nordic design system"
git branch -M main
git remote add origin https://github.com/[SEU-USER]/ronin-crew-site.git
git push -u origin main
```

### 1.3 Estrutura de branches recomendada
```
main          → produção (o que está no ar)
develop       → desenvolvimento ativo
feature/xxx   → features isoladas (ex: feature/prestashop-integration)
```

### 1.4 .gitignore recomendado
Crie o arquivo `.gitignore` na raiz:
```
# dependências
node_modules/
vendor/

# configurações sensíveis
*.env
config/settings_local.php
app/config/parameters.php

# cache PrestaShop
var/cache/
var/logs/
```

---

## FASE 2 — Hostinger: site estático (vitrine HTML)

> **Quando usar:** enquanto o PrestaShop não está pronto, ou para manter a landing page principal em subdomínio separado (ex: `ronincrew.fi` = HTML estático, `shop.ronincrew.fi` = PrestaShop).

### 2.1 Upload via File Manager
1. No hpanel.hostinger.com → **Gerenciador de Arquivos**
2. Navegue até `public_html/`
3. Faça upload de toda a pasta `Site Ronin/` (incluindo subpasta `imagens/`)
4. Certifique que `index.html` está na raiz de `public_html/`

### 2.2 Upload via FTP (mais rápido para muitos arquivos)
Use FileZilla:
- **Host:** `ftp.ronincrew.fi` (ou IP fornecido pelo Hostinger)
- **Usuário / Senha:** credenciais FTP do hpanel
- **Porta:** 21
- Arraste a pasta local para `public_html/`

### 2.3 Conectar domínio
No hpanel → **Domínios** → aponte o domínio para o plano de hospedagem.  
DNS propagação: até 24h (normalmente 1-2h).

### 2.4 SSL gratuito
No hpanel → **SSL** → ativar **Let's Encrypt** para o domínio.  
Força HTTPS: hpanel → **Redirecionar HTTP para HTTPS** → ativar.

---

## FASE 3 — VPS Hostinger: configurar servidor

> **Use a VPS** para rodar o PrestaShop (PHP + MySQL). É mais performático e dá controle total.

### 3.1 Primeiro acesso SSH
```bash
ssh root@[IP-DA-VPS]
# aceitar fingerprint → yes
# digitar senha root
```

### 3.2 Atualizar servidor e instalar stack LAMP
```bash
apt update && apt upgrade -y

# Apache
apt install apache2 -y
systemctl enable apache2
systemctl start apache2

# MySQL
apt install mysql-server -y
mysql_secure_installation
# → Seguir o wizard: definir senha root MySQL, remover usuários anônimos, desabilitar login remoto root

# PHP 8.1 (requerido pelo PrestaShop 8.x)
apt install php8.1 php8.1-cli php8.1-mysql php8.1-curl php8.1-gd \
  php8.1-mbstring php8.1-xml php8.1-zip php8.1-intl php8.1-bcmath \
  libapache2-mod-php8.1 -y

# Confirmar versões
php -v
mysql --version
apache2 -v
```

### 3.3 Configurar MySQL para PrestaShop
```bash
mysql -u root -p

CREATE DATABASE ronincrew_shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'ronin_user'@'localhost' IDENTIFIED BY 'SENHA_FORTE_AQUI';
GRANT ALL PRIVILEGES ON ronincrew_shop.* TO 'ronin_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
> ⚠️ Guarde: `ronincrew_shop` / `ronin_user` / `[sua senha]` — você vai precisar no instalador do PrestaShop.

### 3.4 Configurar VirtualHost Apache
```bash
nano /etc/apache2/sites-available/ronincrew.conf
```
Cole:
```apache
<VirtualHost *:80>
    ServerName ronincrew.fi
    ServerAlias www.ronincrew.fi
    DocumentRoot /var/www/ronincrew

    <Directory /var/www/ronincrew>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ronincrew_error.log
    CustomLog ${APACHE_LOG_DIR}/ronincrew_access.log combined
</VirtualHost>
```
```bash
a2ensite ronincrew.conf
a2enmod rewrite
systemctl reload apache2
```

### 3.5 SSL na VPS (Let's Encrypt)
```bash
apt install certbot python3-certbot-apache -y
certbot --apache -d ronincrew.fi -d www.ronincrew.fi
# Seguir wizard → email → aceitar termos → redirecionar HTTP → HTTPS
```

---

## FASE 4 — PrestaShop: instalação e configuração

### 4.1 Upload do PrestaShop para a VPS
Na sua máquina local:
```bash
# Baixar PrestaShop 8.x
# Descompactar e enviar para a VPS via SCP:
scp -r /caminho/local/prestashop/ root@[IP-VPS]:/var/www/ronincrew/
```
Ou direto na VPS:
```bash
cd /var/www/ronincrew
wget https://github.com/PrestaShop/PrestaShop/releases/download/8.1.7/prestashop_8.1.7.zip
apt install unzip -y
unzip prestashop_8.1.7.zip
chown -R www-data:www-data /var/www/ronincrew
chmod -R 755 /var/www/ronincrew
```

### 4.2 Instalador web
Acesse `https://ronincrew.fi/install` no navegador e siga o wizard:

| Campo | Valor |
|-------|-------|
| Idioma loja | Português (BR) + Finnish + English |
| País | Finland |
| Fuso | Europe/Helsinki |
| Banco de dados host | `localhost` |
| Nome do banco | `ronincrew_shop` |
| Usuário BD | `ronin_user` |
| Senha BD | `[a que você criou]` |
| Prefixo tabelas | `ps_` |
| E-mail admin | `leo.cesargarcia@gmail.com` |

### 4.3 Remover pasta install (obrigatório por segurança)
```bash
rm -rf /var/www/ronincrew/install
```

### 4.4 Configurações essenciais no back-office (`/admin-xxx/`)

**Loja e SEO:**
- Back-office → **Preferências de Loja** → nome: "Ronin Crew", país padrão: Finlândia
- **SEO & URLs** → URLs amigáveis: ativar

**Moeda e idiomas:**
- **Localização** → importar pacote: Finlândia → instala EUR, Finnish, leis locais
- Adicionar idioma PT-BR e EN manualmente ou via pacotes

**Métodos de pagamento:**
- Módulo **Stripe** → instalar e conectar chave do dashboard Stripe
- Módulo **PayPal** → instalar se necessário

**Frete:**
- **Transportadoras** → configurar Posti (Finlândia), DHL, ou integração via módulo

**Impostos:**
- **Taxas** → configurar IVA finlandês: 25.5% (padrão 2024+)

### 4.5 Tema: integrar o design Ronin Crew

**Opção A — Tema filho simples (recomendado para agilidade):**
```bash
# Na VPS
cd /var/www/ronincrew/themes/
mkdir ronin-child
cp -r classic/* ronin-child/
```
Editar CSS e templates do tema para replicar o design do `index.html`.

**Opção B — HTML estático como página inicial (mais rápido):**
- Instalar módulo **"Custom HTML Block"** ou **"Homepage CMS Block"**
- Incorporar o design Ronin Crew como página CMS estática
- PrestaShop cuida do carrinho/checkout; o front usa seu HTML

### 4.6 Criar produtos no PrestaShop

| Produto | SKU | Preço | Variações |
|---------|-----|-------|-----------|
| Ronin Shadow GI Pro | RC-GI-SHADOW-001 | €189 | A0-A5 · Azul |
| Tactical Combat Shorts | RC-NG-TCT-001 | €64 | S-XXL · Preto |
| Ronin Rashguard LS | RC-NG-RG-001 | €59 | S-XXL · Azul |
| Combat Belt — Black Line | RC-BT-BLK-001 | €35 | M1-M4, F |
| Warrior Tee — Oversized | RC-LS-TEE-001 | €45 | S-XXL |
| Arctic GI — Black Edition | RC-GI-ARC-001 | €159 | A0-A5 · Preto |

---

## FASE 5 — Deploy contínuo (GitHub → VPS)

### 5.1 Deploy manual via Git na VPS
```bash
# Na VPS, dentro de /var/www/ronincrew
git clone https://github.com/[SEU-USER]/ronin-crew-site.git .
# Futuras atualizações:
git pull origin main
```

### 5.2 Deploy automático via GitHub Actions (opcional, pós-launch)
Criar `.github/workflows/deploy.yml`:
```yaml
name: Deploy to VPS
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_IP }}
          username: root
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/ronincrew
            git pull origin main
            systemctl reload apache2
```

---

## FASE 6 — Checklist pré-launch

- [ ] `index.html` abre corretamente com `https://`
- [ ] Todas as imagens da pasta `imagens/` carregam (sem 404)
- [ ] Modal de produto abre com galeria de fotos reais
- [ ] Carrinho adiciona e calcula corretamente
- [ ] Formulário de contato envia (Formspree ID configurado)
- [ ] Stripe Payment Link configurado e testado em modo test
- [ ] SSL ativo (cadeado no browser)
- [ ] Google Analytics / Meta Pixel instalado
- [ ] Sitemap.xml gerado (no PrestaShop: SEO → Sitemap)
- [ ] Robots.txt revisado
- [ ] Velocidade: testar em https://pagespeed.web.dev

---

## PRÓXIMA SESSÃO — Pendências com Léo

Quando você enviar os textos e fotos corretos, faremos:

1. **Substituir textos** — hero headline, descrições de produtos, manifesto
2. **Substituir fotos** — trocar Google AI images pelos assets finais
3. **Formspree** — configurar o ID real do formulário
4. **Stripe** — conectar Payment Link real
5. **PrestaShop** — instalar, cadastrar produtos, configurar pagamento e frete
6. **Domínio** — apontar `ronincrew.fi` para a VPS

---

## Referências rápidas

| Recurso | URL |
|---------|-----|
| Hostinger Panel | https://hpanel.hostinger.com |
| GitHub Repo | https://github.com/[SEU-USER]/ronin-crew-site |
| PrestaShop Docs | https://devdocs.prestashop-project.org |
| Stripe Dashboard | https://dashboard.stripe.com |
| Formspree | https://formspree.io/forms |
| Let's Encrypt | https://letsencrypt.org |
| PageSpeed Test | https://pagespeed.web.dev |

---

*Documento gerado em 2026-05 — Ronin Crew OY, Helsinki*
