# Wavlink WL-UG7602HC — Driver no Manjaro Linux

Este repositório contém um tutorial e os arquivos necessários para instalar o driver do **Wavlink WL-UG7602HC** (dock USB com saída de vídeo, chip Silicon Motion SM768) no **Manjaro Linux**.

O instalador oficial foi feito para Ubuntu/Debian e não funciona em distros baseadas em Arch sem ajustes. Este guia resolve isso.

---

## Dispositivo

| Campo       | Info                          |
|-------------|-------------------------------|
| Produto     | Wavlink WL-UG7602HC           |
| Chip        | Silicon Motion SM768          |
| Driver      | SMIUSBDisplay 2.22.1.0        |
| Testado em  | Manjaro Linux, kernel 6.18.33 |

---

## O que está neste repositório

| Arquivo                      | Descrição                                              |
|------------------------------|--------------------------------------------------------|
| `README.md`                  | Este arquivo                                           |
| `INSTALL-MANJARO.md`         | Guia detalhado passo a passo                           |
| `install-manjaro-patched.sh` | Script `install.sh` já corrigido para Manjaro          |
| `SM768-Manjaro-install.zip`  | Zip com todos os arquivos acima + instalador original  |

---

## Instalação rápida

### 1. Instale as dependências

```bash
sudo pacman -S --needed dkms libdrm pkgconf evdi-dkms
```

> O `evdi-dkms` do repositório oficial é compatível com kernels 6.x. A versão bundada no instalador original (1.14.7) não compila em kernels modernos.

### 2. Baixe o instalador oficial

Baixe o driver Linux (arquivo `.run`) no site da Wavlink e coloque em `~/Downloads/`.

> **Atenção:** Este guia e o script patchado foram feitos para a versão **2.22.1.0** do driver. Se você baixar uma versão mais nova, os patches podem precisar ser refeitos.

### 3. Extraia o instalador

```bash
~/Downloads/SMIUSBDisplay-driver.2.22.1.0.run --noexec --target /tmp/smi-driver
```

### 4. Substitua o script de instalação pelo patchado

```bash
cp install-manjaro-patched.sh /tmp/smi-driver/install.sh
```

### 5. Execute

```bash
sudo bash /tmp/smi-driver/install.sh
```

### 6. Habilite o serviço

```bash
echo -e '\n[Install]\nWantedBy=multi-user.target' | sudo tee -a /usr/lib/systemd/system/smiusbdisplay.service
sudo systemctl daemon-reload
sudo systemctl enable --now smiusbdisplay
```

Conecte o monitor USB — pronto.

---

## O que foi alterado no script original

O `install.sh` oficial apresenta três problemas no Manjaro:

1. **Usa `apt`** para verificar e instalar dependências — substituído por `pacman`.
2. **Tenta compilar o EVDI 1.14.7** bundado, que é incompatível com kernels 6.x — substituído pelo uso do `evdi-dkms` do repositório oficial.
3. **Aborta com pedido de reboot** quando detecta o módulo `evdi` carregado — verificação desnecessária removida.

Para o detalhamento completo de cada patch, veja [INSTALL-MANJARO.md](./INSTALL-MANJARO.md).

---

## Verificação

```bash
lsmod | grep evdi               # módulo carregado
systemctl status smiusbdisplay  # serviço ativo
```

---

## Problemas conhecidos

- O instalador oficial não detecta Manjaro como distro suportada e sempre tenta usar `apt`.
- O EVDI bundado (1.14.7) tem APIs incompatíveis com kernels 6.13+ (`MODULE_IMPORT_NS` e `struct_mutex` removido do DRM).
- O arquivo `.service` instalado não tem seção `[Install]`, impedindo o `systemctl enable` sem o patch manual.
