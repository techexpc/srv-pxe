# 🚀 Servidor iPXE Híbrido (srv-pxe)

**Servidor de Deploy e Diagnóstico via Rede (Network Boot Server)**  
*Desenvolvido por Elias Oliveira | Analista de Sistemas*

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

*   **SO:** Ubuntu 22.04+ / Linux Lite 8.0+ (Testado em Dell OptiPlex / Hardware corporativo).
*   **Pacotes:** `nginx`, `dnsmasq`, `ipxe`, `wimtools`, `samba`, `openssh-server`.
*   **Imagens:** ISO do Hiren's BootCD PE, Imagens do Macrium Reflect.

---

## ⚡ Instalação Rápida (Resumo)

> ⚠️ **Atenção:** O guia completo e detalhado com explicações teóricas (RFCs) e troubleshooting está disponível na documentação do projeto. Abaixo, um resumo dos comandos.

### 1. Preparação do Ambiente e IP Fixo
Configure o Netplan para IP estático e pare serviços concorrentes:
```bash
sudo systemctl stop tftpd-hpa atftpd isc-dhcp-server 2>/dev/null
sudo apt-get update && sudo apt-get install -y nginx dnsmasq ipxe ipxe-qemu wimtools samba samba-common-bin
