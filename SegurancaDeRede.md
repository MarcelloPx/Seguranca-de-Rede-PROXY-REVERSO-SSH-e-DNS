# Segurança de Rede — Resumo Técnico

**Autor:** Marcello Paixão da Cruz | **Plataforma:** Alura | **Ano:** 2025

---

## Arquitetura

| Componente | IP | Função |
|---|---|---|
| pfSense | `192.168.1.1` | Firewall de borda |
| Graylog | `172.100.1.101` | SIEM / Logs |
| WAF (NGINX + ModSecurity) | `172.100.2.100` | Proxy reverso |
| Servidor Web | `172.100.2.101` | Backend HTTP |
| DVWA | `172.100.2.102` | App vulnerável para testes |

> Modelo adotado: **Defense in Depth** (Segurança em Camadas)

---

## SSH

Acesso remoto configurado na porta 22 com verificação de fingerprint. O pfSense foi usado como **jump host** para alcançar o Graylog:

```bash
ssh user@172.100.1.101
```

---

## Firewall pfSense

Regras criadas:

- **PORT_ADM** — libera portas 80, 443 e 22 para o Graylog acessar a internet
- **DNS** — libera UDP porta 53 para resolução de nomes
- **DMZ_UPDATE** — permite que Graylog e Servidor Web acessem repositórios
- **WAF → Web** — permite TCP porta 80 do WAF (`172.100.2.100`) para o servidor web (`172.100.2.101`)

---

## IPTables (Graylog)

Segunda camada de firewall no próprio SO:

```bash
# Permitir conexões já estabelecidas
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Liberar acesso ao gateway (pfSense)
iptables -I OUTPUT 1 -d 172.100.1.100 -j ACCEPT

# Bloquear movimentação lateral na DMZ
iptables -I OUTPUT 3 -d 172.100.1.0/24 -j DROP

# Salvar regras
iptables-save > /etc/iptables/rules.v4
```

---

## NGINX + DVWA

- DNS configurado com Cloudflare (`1.1.1.1` / `1.0.0.1`)
- NGINX instalado e habilitado no boot
- DVWA implantado no IP `172.100.2.102`

---

## WAF — Proxy Reverso

Configuração do NGINX como proxy reverso em `/etc/nginx/sites-available/site.lab.conf`:

```nginx
server {
    listen 80;
    server_name site.lab;
    location / {
        proxy_pass http://172.100.2.101/;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Resolução local adicionada no `hosts` do Windows:
```
192.168.15.10  site.lab
```

---

## Testes

| Teste | Resultado |
|---|---|
| Conectividade WAF → Web (`nc -nvz 172.100.2.101 80`) | ✅ Conexão estabelecida |
| SQL Injection (`' OR 1=1 #`) via DVWA | ✅ Bloqueado — HTTP 403 |
| Logs no Graylog (filtro `modSecurity`) | ✅ Evento registrado |

---

## Conclusão

O ambiente demonstrou eficácia da segurança em camadas: SSH com jump host, pfSense com regras granulares, IPTables no SO, proxy reverso NGINX e ModSecurity bloqueando SQL Injection, com tudo auditável via Graylog.
