# SM768 USB Display Driver — Manjaro Linux

Guia de instalação do driver Silicon Motion SM768 (SMIUSBDisplay 2.22.1.0) no Manjaro Linux.

> O instalador oficial foi feito para Ubuntu/Debian e usa `apt`. Este guia corrige a instalação para Manjaro (Arch-based).

---

## Pré-requisitos

- Kernel 6.x (testado no 6.18.33-1-MANJARO)
- `sudo` configurado

---

## Passo 1 — Instalar dependências

```bash
sudo pacman -S --needed dkms libdrm pkgconf evdi-dkms
```

> `evdi-dkms` é o módulo de kernel necessário. A versão do repositório oficial (1.14.11+) é compatível com kernels 6.x, diferente da versão bundada no instalador (1.14.7).

---

## Passo 2 — Extrair o instalador

```bash
~/Downloads/SM768\ Driver-20260409/Linux/SMIUSBDisplay-driver.2.22.1.0.run --noexec --target /tmp/smi-driver
cd /tmp/smi-driver
```

---

## Passo 3 — Corrigir o script de instalação

O `install.sh` usa `apt` para verificar dependências e tenta compilar uma versão antiga do EVDI incompatível com kernels 6.x. Aplique os patches abaixo:

### 3.1 — Corrigir checagem de dependências

Substitua as funções `check_dkms`, `check_libdrm` e `check_pkg`:

```bash
sed -i 's/apt list -qq --installed dkms.*grep -q dkms/hash dkms 2>\/dev\/null/' install.sh
```

Ou edite manualmente o `install.sh` trocando os três blocos:

**Antes:**
```bash
check_dkms()
{
  #hash apt 2>/dev/null || return
  apt list -qq --installed dkms 2>/dev/null | grep -q dkms
}

check_libdrm()
{
  #hash apt 2>/dev/null || return
  apt list -qq --installed libdrm-dev 2>/dev/null | grep -q libdrm-dev
}

check_pkg()
{
  #hash apt 2>/dev/null || return
  apt list -qq --installed pkg-config 2>/dev/null | grep -q pkg-config
}
```

**Depois:**
```bash
check_dkms()
{
  hash dkms 2>/dev/null
}

check_libdrm()
{
  pacman -Qi libdrm &>/dev/null
}

check_pkg()
{
  hash pkg-config 2>/dev/null || hash pkgconf 2>/dev/null
}
```

E substitua `install_dependencies_apt`:

**Antes:**
```bash
install_dependencies_apt()
{
  echo "[ Dependency check ]"
  #hash dkms 2>/dev/null
  apt list -qq --installed dkms 2>/dev/null | grep -q dkms
  local install_dkms=$?
  apt list -qq --installed libdrm-dev 2>/dev/null | grep -q libdrm-dev
  local install_libdrm=$?

  apt list -qq --installed pkg-config 2>/dev/null | grep -q pkg-config
  local install_pkg=$?

  if [ "$install_dkms" != 0 ] || [ "$install_libdrm" != 0 ] || [ "$install_pkg" != 0 ]; then
    echo "[ Installing dependencies ]"
    apt_ask_for_dependencies || (apt_ask_for_update && apt_ask_for_dependencies) || check_requirements
    read -rp 'Do you want to continue? [Y/n] ' CHOICE
    [[ "${CHOICE:-Y}" == "${CHOICE#[Yy]}" ]] && exit 0

    apt install -y dkms libdrm-dev pkg-config || check_requirements
  fi
}
```

**Depois:**
```bash
install_dependencies_apt()
{
  echo "[ Dependency check ]"
  check_dkms
  local install_dkms=$?
  check_libdrm
  local install_libdrm=$?
  check_pkg
  local install_pkg=$?

  if [ "$install_dkms" != 0 ] || [ "$install_libdrm" != 0 ] || [ "$install_pkg" != 0 ]; then
    echo "[ Installing dependencies ]"
    pacman -S --needed --noconfirm dkms libdrm pkgconf || check_requirements
  fi
}
```

### 3.2 — Usar o EVDI do sistema em vez do bundado

Substitua a função `install_evdi` inteira:

**Antes:**
```bash
install_evdi()
{
  TARGZ="$1"
  ERRORS="$2"
  ...
  dkms install "${EVDI}/module"
  ...
  make  # compila libevdi da fonte antiga
  ...
}
```

**Depois:**
```bash
install_evdi()
{
  ERRORS="$2"
  local EVDI_DRM_DEPS

  echo "[[ Using system-installed EVDI (evdi-dkms) ]]"

  echo "[[ Installing module configuration files ]]"
  printf '%s\n' 'evdi' > /etc/modules-load.d/evdi.conf

  printf '%s\n' 'options evdi initial_device_count=4' \
        > /etc/modprobe.d/evdi.conf
  EVDI_DRM_DEPS=$(sed -n -e '/^drm_kms_helper/p' /proc/modules | awk '{print $4}' | tr ',' ' ')
  EVDI_DRM_DEPS=${EVDI_DRM_DEPS/evdi/}

  [[ "${EVDI_DRM_DEPS}" ]] && printf 'softdep %s pre: %s\n' 'evdi' "${EVDI_DRM_DEPS}" \
        >> /etc/modprobe.d/evdi.conf

  echo "[[ Backuping EVDI DKMS module ]]"
  local EVDI_VERSION
  EVDI_VERSION=$(ls -t /usr/src | grep evdi | head -n1)
  cp -rf /usr/src/$EVDI_VERSION $COREDIR/module
  cp /etc/modprobe.d/evdi.conf $COREDIR

  echo "[[ Using system EVDI library ]]"
  cp -f /usr/lib/libevdi.so "$COREDIR/libevdi.so" || { echo "Failed to copy libevdi.so" > "$ERRORS"; return 1; }
  chmod 0755 "$COREDIR/libevdi.so"
  ln -sf "$COREDIR/libevdi.so" /usr/lib/libevdi.so.0
  ln -sf "$COREDIR/libevdi.so" /usr/lib/libevdi.so.1
}
```

### 3.3 — Remover verificação de reboot desnecessária

Localize o bloco abaixo e substitua:

**Antes:**
```bash
  modprobe evdi

  if [ -f /sys/devices/evdi/count ]; then
    echo "WARNING: EVDI kernel module is already running." >&2

    if [ -d $COREDIR ]; then
      echo "Please uninstall all other versions of $PRODUCT before attempting to install." >&2
      echo "Installation terminated." >&2
      exit 1
    elif [ -d $OTHERPDDIR ]; then
      ...
    else
      echo "Please reboot before attempting to re-install $PRODUCT." >&2
      echo "Installation terminated." >&2
      exit 1
    fi
  fi
}
```

**Depois:**
```bash
  modprobe evdi
}
```

---

## Passo 4 — Executar o instalador corrigido

```bash
sudo bash /tmp/smi-driver/install.sh
```

Saída esperada ao final:
```
Installation complete!
```

---

## Passo 5 — Habilitar e iniciar o serviço

O arquivo de serviço instalado não tem seção `[Install]`, então é necessário adicioná-la antes de habilitar:

```bash
echo -e '\n[Install]\nWantedBy=multi-user.target' | sudo tee -a /usr/lib/systemd/system/smiusbdisplay.service
sudo systemctl daemon-reload
sudo systemctl enable smiusbdisplay
sudo systemctl start smiusbdisplay
```

Verifique se está rodando:

```bash
systemctl status smiusbdisplay
```

O serviço deve aparecer como `active (running)`.

---

## Verificação

```bash
lsmod | grep evdi               # módulo carregado
systemctl status smiusbdisplay  # serviço rodando
```

Conecte o monitor USB — ele deve ser reconhecido automaticamente.

---

## Notas

- Não é necessário reiniciar após a instalação.
- O serviço sobe automaticamente no boot após o `systemctl enable`.
- Testado em: Manjaro Linux, kernel 6.18.33-1-MANJARO, driver SMIUSBDisplay 2.22.1.0.
