# Havit Fuxi H3 no Linux Mint — Correção de áudio em apenas um lado

## Descrição

Este guia documenta a correção de problemas de áudio do headset Havit Fuxi H3 utilizando o dongle USB 2.4 GHz no Linux Mint.

O problema identificado era a reprodução de áudio em apenas um lado do headset, apesar de os dois alto-falantes funcionarem normalmente no Windows 10 e no celular.

### Dispositivo testado

- Headset: Havit Fuxi H3
- Fabricante USB: Weltrend Semiconductor
- Identificador USB: `040b:0897`
- Driver: `snd_usb_audio`
- Servidor de áudio: PipeWire
- Gerenciador de sessão: WirePlumber
- Sistema: Linux Mint baseado em Ubuntu 24.04

## 1. Identificação do dispositivo

Execute:

```bash
lsusb | grep -iE '040b|Weltrend|Fuxi'
```

Resultado esperado:

```text
ID 040b:0897 Weltrend Semiconductor Fuxi-H3
```

## 2. Identificação do problema

Verifique os controles ALSA:

```bash
amixer -c 3 scontrols
amixer -c 3 scontents
```

No equipamento testado, foram encontrados dois controles PCM:

```text
PCM,0
PCM,1
```

O controle PCM secundário apresentava volume de 0%, indicando uma possível relação com o problema dos canais.

## 3. Correção manual

Execute:

```bash
amixer -c 3 sset 'PCM',0 100% unmute
amixer -c 3 sset 'PCM',1 100% unmute
```

Confira:

```bash
amixer -c 3 scontents
```

Os dois controles devem permanecer em 100%.

Observação: o número da placa pode mudar entre computadores. Confira em `/proc/asound/cards` antes de executar.

## 4. Configuração permanente do WirePlumber

Crie o diretório:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/
```

Crie o arquivo:

```bash
nano ~/.config/wireplumber/wireplumber.conf.d/51-fuxi-h3-fix.conf
```

Adicione:

```ini
monitor.alsa.rules = [
  {
    matches = [
      {
        device.vendor.id = "0x040b"
        device.product.id = "0x0897"
      }
    ]
    actions = {
      update-props = {
        api.alsa.soft-mixer = true
        api.alsa.ignore-dB = true
        api.alsa.enable-hw-volume = false
        api.alsa.headroom = 0
      }
    }
  }
]
```

Salve o arquivo.

Reinicie os serviços:

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
```

Essa configuração direciona o controle de volume para o software, evitando alterações diretas nos controles físicos do headset.

## 5. Script para manter o volume PCM em 100%

Crie:

```bash
sudo nano /usr/local/bin/fuxi-h3-volume-fix.sh
```

Adicione:

```bash
#!/bin/bash

sleep 2

CARD=$(grep -iE 'FuxiH3|Fuxi-H3' /proc/asound/cards 2>/dev/null | head -1 | awk '{print $1}')

if [ -z "$CARD" ]; then
    exit 0
fi

/usr/bin/amixer -c "$CARD" sset 'PCM',0 100%,100% unmute
/usr/bin/amixer -c "$CARD" sset 'PCM',1 100% unmute
```

Dê permissão:

```bash
sudo chmod +x /usr/local/bin/fuxi-h3-volume-fix.sh
```

## 6. Aplicação automática ao conectar o dongle

Crie a regra:

```bash
sudo nano /etc/udev/rules.d/99-fuxi-h3.rules
```

Adicione:

```udev
ACTION=="add", SUBSYSTEM=="sound", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/local/bin/fuxi-h3-volume-fix.sh"
```

Recarregue as regras:

```bash
sudo udevadm control --reload-rules
```

Aplique a correção imediatamente:

```bash
/usr/local/bin/fuxi-h3-volume-fix.sh
```

## 7. Reiniciar o áudio

Execute:

```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
```

Desconecte e conecte novamente o dongle USB.

## 8. Verificação

Confira os controles PCM:

```bash
CARD=$(grep -iE 'FuxiH3|Fuxi-H3' /proc/asound/cards | head -1 | awk '{print $1}')
amixer -c "$CARD" scontents
```

Confira os dispositivos de áudio:

```bash
pactl list sinks short
wpctl status
```

## 9. Desinstalação

Para remover as configurações:

```bash
rm -f ~/.config/wireplumber/wireplumber.conf.d/51-fuxi-h3-fix.conf
sudo rm -f /usr/local/bin/fuxi-h3-volume-fix.sh
sudo rm -f /etc/udev/rules.d/99-fuxi-h3.rules
sudo udevadm control --reload-rules
systemctl --user restart wireplumber pipewire pipewire-pulse
```

## 10. Compatibilidade

| Conexão | Aplicação |
|---|---|
| Dongle USB 2.4 GHz | Testado |
| Bluetooth | Não testado |
| USB-C | Não testado |
| P2 3,5 mm | Não se aplica |

## 11. Referências

- https://github.com/linuxmint/linuxmint/issues/822
- https://github.com/hFerbic/havitsolution
- https://wiki.archlinux.org/title/PipeWire/Troubleshooting

## 12. Histórico de testes

No computador utilizado para o diagnóstico:

- O headset funcionava normalmente no Windows 10.
- O headset funcionava normalmente no celular.
- O Linux reconhecia o dispositivo USB `040b:0897`.
- Foram identificados dois controles PCM.
- O controle PCM secundário estava inicialmente em 0%.
- Após a configuração do WirePlumber e o ajuste dos dois controles PCM para 100%, o áudio voltou a funcionar nos dois lados.

**Resultado: problema resolvido no equipamento testado.**

> Comando único de instalação


```bash


#!/bin/bash
set -e

echo "=========================================="
echo " Havit Fuxi H3 - Correcao de audio Linux"
echo "=========================================="

VENDOR="040b"
PRODUCT="0897"
WP_DIR="$HOME/.config/wireplumber/wireplumber.conf.d"

# 1. Verificar dispositivo USB
if ! lsusb | grep -qi "${VENDOR}:${PRODUCT}"; then
    echo "ERRO: Havit Fuxi H3 nao encontrado."
    echo "Conecte o dongle USB e tente novamente."
    exit 1
fi

echo "[OK] Havit Fuxi H3 identificado."

# 2. Criar configuracao WirePlumber
mkdir -p "$WP_DIR"

cat > "$WP_DIR/51-fuxi-h3-fix.conf" <<'EOF'
monitor.alsa.rules = [
  {
    matches = [
      {
        device.vendor.id = "0x040b"
        device.product.id = "0x0897"
      }
    ]
    actions = {
      update-props = {
        api.alsa.soft-mixer = true
        api.alsa.ignore-dB = true
        api.alsa.enable-hw-volume = false
        api.alsa.headroom = 0
      }
    }
  }
]
EOF

echo "[OK] Configuracao WirePlumber criada."

# 3. Criar script de correcao PCM
sudo tee /usr/local/bin/fuxi-h3-volume-fix.sh > /dev/null <<'EOF'
#!/bin/bash

for i in {1..10}; do
    CARD=$(awk '
        /FuxiH3|Fuxi-H3/ {
            print $1
            exit
        }
    ' /proc/asound/cards 2>/dev/null)

    if [ -n "$CARD" ]; then
        break
    fi

    sleep 1
done

if [ -z "$CARD" ]; then
    echo "Havit Fuxi H3 nao encontrado no ALSA."
    exit 1
fi

echo "Havit Fuxi H3 encontrado na placa ALSA $CARD."

amixer -c "$CARD" sset 'PCM',0 100% unmute
amixer -c "$CARD" sset 'PCM',1 100% unmute

echo "Controles PCM ajustados para 100%."
EOF

sudo chmod +x /usr/local/bin/fuxi-h3-volume-fix.sh

echo "[OK] Script de correcao instalado."

# 4. Criar regra automatica udev
sudo tee /etc/udev/rules.d/99-fuxi-h3.rules > /dev/null <<'EOF'
ACTION=="add", SUBSYSTEM=="sound", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/bin/systemd-run --no-block --quiet /usr/local/bin/fuxi-h3-volume-fix.sh"
EOF

sudo udevadm control --reload-rules

echo "[OK] Regra automatica USB instalada."

# 5. Aplicar correcao imediatamente
echo "Aplicando ajuste dos controles PCM..."

sudo /usr/local/bin/fuxi-h3-volume-fix.sh

# 6. Reiniciar servicos de audio
echo "Reiniciando PipeWire e WirePlumber..."

systemctl --user restart wireplumber
systemctl --user restart pipewire
systemctl --user restart pipewire-pulse

echo ""
echo "=========================================="
echo " INSTALACAO CONCLUIDA!"
echo "=========================================="
echo ""
echo "Desconecte e conecte novamente o dongle USB."
echo "Depois teste os dois lados do headset."
echo ""
echo "Para verificar:"
echo "amixer -c NUMERO_DA_PLACA scontents"
echo "wpctl status"
echo "pactl list sinks short"


```


## Contribuições

Contribuições são bem-vindas! Informe a distribuição Linux, versão do kernel, PipeWire, WirePlumber e os resultados dos testes PCM.

O objetivo é documentar uma solução reproduzível para usuários do Havit Fuxi H3 no Linux.