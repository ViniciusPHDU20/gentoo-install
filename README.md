# gentoo-install — Sovereign Edition

> **Fork de [oddlama/gentoo-install](https://github.com/oddlama/gentoo-install)**  
> Mantido por [ViniciusPHDU20](https://github.com/ViniciusPHDU20)

Este fork estende o instalador original com suporte a **F2FS**, detecção automática de **NVMe** e uma arquitetura em dois estágios que separa a instalação base do ambiente de desktop.

---

## O que há de novo neste fork

| Recurso | Original | Sovereign Edition |
|---------|----------|------------------|
| Sistemas de arquivo suportados | ext4, btrfs, ZFS | ext4, btrfs, ZFS, **F2FS** ✨ |
| Detecção automática de NVMe | ❌ | ✅ Sugere F2FS automaticamente |
| Pós-instalação (Hyprland + JaKooLit) | ❌ | ✅ Script separado (veja abaixo) |
| Instalação base | Source-based | Source-based (+ binpkg opcional) |

---

## Arquitetura do projeto

```
gentoo-install/          ← Este repositório
├─ configure             ← TUI de configuração (menuconfig-style)
├─ install               ← Instalador base Gentoo
├─ gentoo.conf.example   ← Exemplo comentado de configuração
└─ scripts/              ← Utilitários internos

Sovereign_JaKooLit_Gentoo.sh  ← Pós-instalação SEPARADO
                               (Hyprland + dotfiles JaKooLit)
                               Executado APÓS a base ser validada
```

### Por que a separação?

O instalador base instala um sistema **Gentoo mínimo** funcional — particionamento, portage, kernel, rede. O script de pós-instalação (`Sovereign_JaKooLit_Gentoo.sh`) é independente e voltado especificamente para quem deseja o ambiente **Hyprland com dotfiles do JaKooLit**. Ele só deve ser executado após a instalação base ser **confirmada sem erros**.

---

## Uso rápido

### 1. Boot em um live environment

Recomendado: [Arch Linux ISO](https://www.archlinux.org/download/) (o instalador consegue baixar dependências automaticamente).

```bash
pacman -Sy git
git clone https://github.com/ViniciusPHDU20/gentoo-install
cd gentoo-install
```

### 2. Configurar

```bash
./configure     # Abre o menu TUI — configure e salve como gentoo.conf
```

> **💡 Dica NVMe:** Se o seu disco raiz for NVMe (ex: `/dev/nvme0n1`), selecione `root_fs=f2fs` no menu de particionamento. F2FS foi projetado especificamente para flash storage e oferece melhor desempenho e vida útil em SSDs NVMe.

**Exemplo de configuração rápida para NVMe com F2FS:**
```bash
# Em gentoo.conf (ou gerado pelo ./configure):
function disk_configuration() {
    create_classic_single_disk_layout swap=8GiB type="efi" luks=false root_fs=f2fs /dev/nvme0n1
}
```

**Exemplo com ext4 (HDs convencionais):**
```bash
function disk_configuration() {
    create_classic_single_disk_layout swap=8GiB type="efi" luks=true root_fs=ext4 /dev/sda
}
```

### 3. Instalar a base

```bash
./install       # Inicia a instalação — não necessita supervisão após o particionamento
```

A instalação realiza (em ordem):

1. Particionamento do disco
2. Download e verificação criptográfica do stage3
3. *(dentro do chroot)*
4. Configuração do Portage (rsync/git sync, mirrors)
5. Configuração base (hostname, timezone, keymap, locales)
6. Instalação de pacotes essenciais (git, kernel…)
7. Sistema bootável (fstab, initramfs, entrada EFI via efibootmgr)
8. Sistema mínimo funcional (rede cabeada, eix, senha root)
   - *(Opcional)* sshd com configuração segura
   - *(Opcional)* pacotes adicionais do `gentoo.conf`

### 4. *(Opcional)* Pós-instalação — Hyprland + JaKooLit

> ⚠️ **Este passo só deve ser executado após a instalação base ser concluída e validada sem erros.**

O script de pós-instalação é mantido em repositório separado e instala:
- **Hyprland** (Wayland compositor)
- **Dotfiles do JaKooLit** (Waybar, Rofi, themes, etc.)
- Pacotes complementares do ambiente desktop

```bash
# Após reboot no sistema Gentoo instalado:
bash Sovereign_JaKooLit_Gentoo.sh
```

---

## Sistemas de arquivo disponíveis

| FS | Melhor para | Notas |
|----|-------------|-------|
| `ext4` | HDs, SSDs SATA | Estável, amplamente suportado |
| `btrfs` | SSDs, snapshots | Compressão transparente, subvolumes |
| `f2fs` | **NVMe, SSDs** ✨ | Otimizado para flash, melhor performance |
| `ZFS` | Servidores, múltiplos discos | Snapshots, RAID, compressão |

### F2FS em NVMe — por que usar?

F2FS (Flash-Friendly File System) foi desenvolvido pela Samsung especificamente para storage flash. Em NVMe, oferece:

- **Menor write amplification** → maior vida útil do SSD
- **Melhor throughput sequencial** em operações de I/O
- **Latência reduzida** em workloads mistos (compilação, gaming)
- Suporte nativo no kernel Linux desde 3.8

**Pré-requisito:** certifique-se de que `sys-fs/f2fs-tools` está disponível no live environment:
```bash
# Arch Linux live:
pacman -S f2fs-tools

# Gentoo (dentro do chroot):
emerge -av sys-fs/f2fs-tools
```

---

## Kernel: source ou binário?

O instalador suporta dois modos (configurável via `KERNEL_TYPE`):

| Modo | Variável | Descrição |
|------|----------|-----------|
| **Binário** | `KERNEL_TYPE=bin` | Usa `gentoo-kernel-bin` — rápido, funciona em todo hardware comum |
| **Source** | `KERNEL_TYPE=source` | Compila `gentoo-kernel` — mesmo config do binário, mas local |

> **Recomendação:** comece com `bin` para validar a instalação. Compile um kernel customizado depois do primeiro boot para otimizar para seu hardware específico.

Para atualizar o kernel após a instalação:

```bash
# 1. Emerge novo kernel
emerge -av sys-kernel/gentoo-kernel-bin

# 2. Selecionar versão
eselect kernel set <kver>

# 3. Backup
mv "$kernel"{,.bak}; mv "$initrd"{,.bak}

# 4. Novo initramfs
generate_initramfs.sh <kver> "$initrd"

# 5. Copiar kernel
cp /boot/kernel-<kver> "$kernel"    # systemd
# ou
cp /boot/vmlinuz-<kver> "$kernel"  # openrc
```

---

## Configuração avançada

Todas as opções estão documentadas em [`gentoo.conf.example`](gentoo.conf.example) e nos menus de ajuda do TUI.

### Otimizações pós-instalação recomendadas

```bash
# 1. Ler notícias do Portage
eselect news read

# 2. Otimizar CFLAGS para o seu CPU
emerge -av app-misc/resolve-march-native
resolve-march-native | tee -a /etc/portage/make.conf

# 3. Detectar flags de CPU
emerge -av app-portage/cpuid2cpuflags
cpuid2cpuflags >> /etc/portage/make.conf

# 4. Umask seguro
echo 'umask 077' >> /etc/profile
```

### SSH (opcional)

O instalador pode configurar um sshd com segurança elevada:
- Apenas chaves **ed25519**
- Algoritmos de troca de chave restritos
- **Sem autenticação por senha**
- Apenas root pode logar

Forneça sua chave pública em `ROOT_SSH_AUTHORIZED_KEYS` no `gentoo.conf`.

---

## Troubleshooting

### OOM durante compilação (cc1plus killed)

Se a compilação parar com `Out of memory: Killed process ... (cc1plus)`, o sistema ficou sem RAM durante a compilação paralela (comum com `-j16` ou mais em pacotes grandes como Node.js ou Firefox).

**Solução:** criar um swapfile de emergência:
```bash
fallocate -l 12G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Retome a compilação — o Portage continuará de onde parou:
```bash
emerge --resume
```

### blkid não encontra UUID após particionamento

Certifique-se de que todos os dispositivos estão desmontados antes de iniciar:
```bash
wipefs -a <DEVICE>
```

### ZFS requer kernel mais novo

```bash
# No shell de emergência dentro do chroot (pressione S<Enter>):
echo 'ACCEPT_KEYWORDS="~amd64"' >> /etc/portage/make.conf
emerge -v gentoo-kernel-bin
exit
# Selecione 'retry'
```

---

## Referências

- [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64)
- [Sakaki's EFI Install Guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide)
- [F2FS — Gentoo Wiki](https://wiki.gentoo.org/wiki/F2FS)
- [JaKooLit Hyprland dotfiles](https://github.com/JaKooLit/Arch-Hyprland)
- [Projeto upstream — oddlama/gentoo-install](https://github.com/oddlama/gentoo-install)

---

## Licença

Este fork mantém a licença original do projeto upstream. Veja [LICENSE](LICENSE).
