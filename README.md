# 🚀 Servidor iPXE Híbrido (srv-pxe)
**Servidor de Deploy e Diagnóstico via Rede (Network Boot Server)**  
*Desenvolvido por Elias Oliveira | Analista de Sistemas*
Projeto completo: https://github.com/techexpc/srv-pxe/blob/main/Documento%20de%20Arquitetura%20e%20Implanta%C3%A7%C3%A3o%20vPROJETO.pdf
<img width="1672" height="941" alt="iPXE - 4" src="https://github.com/user-attachments/assets/146bf20f-aa70-40f0-9d58-494604f4fc2d" />

![Linux](https://img.shields.io/badge/OS-Linux%20Lite%20/%20Ubuntu-blue)
![iPXE](https://img.shields.io/badge/Boot-iPXE-green)
![Nginx](https://img.shields.io/badge/Web-Nginx-orange)
![Samba](https://img.shields.io/badge/File%20Sharing-Samba%20(SMB)-purple)
![License](https://img.shields.io/badge/License-MIT-yellow)

## 📖 Visão Geral

O **srv-pxe** é uma solução corporativa de inicialização via rede que supera as limitações do PXE tradicional. Enquanto o PXE clássico depende exclusivamente do lento protocolo TFTP, este projeto utiliza uma **Arquitetura Híbrida**: o TFTP é usado apenas para o bootstrap inicial (Fase 1), e o **HTTP** assume o controle para a transferência de imagens pesadas (Fase 2), reduzindo o tempo de boot de minutos para segundos.

### 🎯 Casos de Uso
*   Implantação de Sistemas Operacionais (OS Deployment).
*   Diagnóstico e Rescue (Hiren's BootCD PE).
*   Backup e Restauração de Imagens (Macrium Reflect).
*   Ambientes de Disaster Recovery e Laboratórios.
<img width="1672" height="941" alt="Funcionamento _ 1 srvipxe" src="https://github.com/user-attachments/assets/698653b4-b534-40fd-bb18-9ec9fee0807b" />

---

## 🏗️ Arquitetura da Solução

A infraestrutura foi desenhada para **zero impacto** no DHCP corporativo, utilizando a técnica de **ProxyDHCP**.

1. **Estação Cliente:** Envia DHCP Discover.
2. **DHCP Corporativo:** Fornece IP, Máscara, Gateway e DNS.
3. **Dnsmasq (ProxyDHCP):** Fornece apenas o `Next-Server` e `Boot-Filename`.
4. **TFTP (Fase 1):** Entrega o binário iPXE (`undionly.kpxe` ou `ipxe.efi`).
5. **iPXE (Fase 2):** Carrega o menu dinâmico (`boot.ipxe`) via **HTTP (Nginx)**.
6. **WIMBoot:** Baixa `boot.wim`, `BCD` e `boot.sdi` via HTTP e monta na RAM.
7. **Samba (SMB):** Conecta os compartilhamentos de backup/imagens (com suporte a NTLMv1 para WinPE).

---

## 🛠️ Pré-requisitos

*   **SO:** Ubuntu 22.04+ / Linux Lite 8.0+ (https://www.linuxliteos.com/pt-br/) Testado em Dell OptiPlex / Hardware corporativo.
*   **Pacotes:** `nginx`, `dnsmasq`, `ipxe`, `wimtools`, `samba`, `openssh-server`.
*   **Imagens:** ISO do Hiren's BootCD PE, Imagens do Macrium Reflect.

---

## ⚡ Instalação Rápida (Resumo)

> ⚠️ **Atenção:** O guia completo e detalhado com explicações teóricas (RFCs) e troubleshooting está disponível na documentação do projeto. Abaixo, um resumo dos comandos.

### 1. Preparação do Ambiente e IP Fixo
Configure o Netplan para IP estático e pare serviços concorrentes:
```
sudo systemctl stop tftpd-hpa atftpd isc-dhcp-server 2>/dev/null
sudo apt-get update && sudo apt-get install -y nginx dnsmasq ipxe ipxe-qemu wimtools samba samba-common-bin
```
### 2. Estrutura de Diretórios e Binários
```bash
sudo mkdir -p /tftpboot /var/www/html/ipxe/hirens /var/www/html/ipxe/macrium
sudo cp /usr/lib/ipxe/ipxe.efi /tftpboot/
sudo cp /usr/lib/ipxe/undionly.kpxe /tftpboot/
sudo wget -O /var/www/html/ipxe/wimboot https://github.com/ipxe/wimboot/releases/latest/download/wimboot
```
### 3. Configuração do ProxyDHCP (Dnsmasq)
Edite /etc/dnsmasq.conf:
```ini
port=0
dhcp-range=10.64.0.0,proxy
enable-tftp
tftp-root=/tftpboot
pxe-service=x86PC, "Boot Legacy BIOS", undionly.kpxe
pxe-service=X86-64_EFI, "Boot UEFI PXE", ipxe.efi
```
4. Injeção de Ferramentas no WIM (Heren's)
Utilize o wimtools para injetar ferramentas portáteis diretamente na imagem do WinPE:
```bash
sudo wimlib-imagex update /var/www/html/ipxe/hirens/boot.wim 1 <<EOF
add /home/srvipxe/Macrium /Programs/Macrium
add /home/srvipxe/Putty /Programs/Putty
EOF
```
5. Menu iPXE (/var/www/html/ipxe/boot.ipxe)
```ipxe
#!ipxe
dhcp
set boot-url http://10.64.0.80/ipxe
menu Central de Diagnostico e Deploy - Host: srv-pxe
item hirens   [1] Hiren's BootCD PE (Injetado)
item macrium  [2] Macrium Reflect (Deploy)
choose target && goto ${target}

:hirens
kernel ${boot-url}/wimboot
initrd ${boot-url}/hirens/bcd bcd
initrd ${boot-url}/hirens/boot.sdi boot.sdi
initrd ${boot-url}/hirens/boot.wim boot.wim
boot
```
🛡️ Troubleshooting e QA
Erro de Mapeamento SMB no WinPE (System error 86)
Ambientes WinPE legados exigem NTLMv1. Se o mapeamento de rede falhar, abra o CMD (Shift + F10) e force a limpeza de cache e o mapeamento com domínio local:
```cmd
net use * /delete /y
net use Z: \\10.64.0.80\Macrium_Restrito /user:.\srvipxebkp SUA_SENHA
```
Checklist de Validação (QA)

* dnsmasq --test retorna "syntax check OK".
* Nginx serve o script boot.ipxe via HTTP.
* Samba permite acesso anônimo no Macrium_Publico e restrito no Macrium_Restrito.
* Cliente UEFI e Legacy iniciam o menu iPXE corretamente.

💾 Backup e Restauração
Para salvar o estado funcional do servidor (configurações, TFTP, Nginx e WIMs):
```bash
sudo tar -czvf /home/srvipxe/backup_srv_pxe_funcional.tar.gz \
  /etc/netplan/ /etc/dnsmasq.conf /etc/samba/smb.conf /tftpboot/ /var/www/html/ipxe/
```
Restaurar:
```bash
sudo tar -xzvf /home/srvipxe/backup_srv_pxe_funcional.tar.gz -C /
sudo systemctl restart nginx dnsmasq smbd nmbd
```
📚 Fundamentação Teórica e Referências
Este projeto foi fundamentado em padrões da indústria e RFCs do IETF:

* RFC 2131: Dynamic Host Configuration Protocol (DHCP).
* RFC 4578: DHCP Options for Intel PXE.
* RFC 1350: The TFTP Protocol.
* iPXE Open Source Network Boot Firmware
* Wimlib - Open source Windows Imaging

🤝 Contribuições e Licença
Projetos de infraestrutura aberta são essenciais para a evolução do Service Desk e Suporte Corporativo. Contribuições, forks e issues são bem-vindos!

* Autor: Elias Oliveira
* Linkedin: https://www.linkedin.com/in/elias-analistatecnico
* Contato: oliveira.expc@gmail.com
* Licença: MIT License
