# battery-warn

Aviso de batería baja para Linux. Muestra una notificación cuando la batería baja de ciertos porcentajes por defecto 20%, 10% y 5% y el equipo no está conectado al cargador.

Está pensado para entornos como Hyprland, dwm, i3, etc. donde no hay un indicador de batería integrado y es fácil no darse cuenta de que se está agotando. Funciona igual en cualquier escritorio porque solo depende de tres cosas: el kernel, el estándar de notificaciones de freedesktop.org y systemd.

Probado en Arch Linux.

## Cómo funciona

```
kernel (sysfs) -> script bash -> notify-send -> D-Bus -> demonio de notificaciones -> pantalla
                      ^
          servicio de usuario de systemd
```

1. El script lee `/sys/class/power_supply/BAT*/capacity` y `status` cada `INTERVAL` segundos.
2. Si el estado es `Discharging` y el porcentaje está en o por debajo de un umbral, envía una notificación con `notify-send`.
3. Cada umbral avisa una sola vez por descarga. Cuando se conecta el cargador, el estado se reinicia.
4. Un servicio de usuario de systemd mantiene el script corriendo desde que inicias sesión, sin importar el entorno gráfico.

Si el equipo no tiene batería (PC de escritorio), el script termina sin error.

## Archivos

| Archivo | Descripción |
|---|---|
| `battery-warn` | Script en bash que lee la batería y envía los avisos |
| `battery-warn.service` | Unidad de usuario de systemd que ejecuta el script |

## Requisitos

- Linux con systemd
- Bash
- `libnotify` (proporciona `notify-send`)
- Un demonio de notificaciones en ejecución

En Arch Linux:

```bash
sudo pacman -S libnotify
```

### Demonio de notificaciones

`notify-send` solo envía el mensaje. Otro programa debe recibirlo y dibujarlo. Si no hay ninguno, el comando falla con `The name is not activatable`.

| Entorno | Demonio | Instalación |
|---|---|---|
| Hyprland (Wayland) | `dunst` o `mako` | `sudo pacman -S dunst` |
| dwm, i3 (X11) | `dunst` | `sudo pacman -S dunst` |
| XFCE | `xfce4-notifyd` | Ya viene incluido |

Comprueba que funciona antes de continuar:

```bash
notify-send "prueba" "hola"
```


#### Iniciar el demonio con la sesión

**Hyprland 0.55 o superior **, en `~/.config/hypr/hyprland.lua`:

```lua
hl.on("hyprland.start", function()
    hl.exec_cmd("dunst")
end)
```

**dwm**, en `~/.xinitrc`, antes de `exec dwm`:

```bash
dunst &
```

Si usas un display manager en lugar de `startx`, el archivo equivalente suele ser `~/.xprofile`.


## Instalación

```bash
git clone https://github.com/USUARIO/battery-warn.git
cd battery-warn

mkdir -p ~/.local/bin ~/.config/systemd/user
cp battery-warn ~/.local/bin/battery-warn
chmod +x ~/.local/bin/battery-warn
cp battery-warn.service ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable --now battery-warn.service
```


Verifica que está corriendo:

```bash
systemctl --user status battery-warn.service
```


## Configuración

Edita `~/.local/bin/battery-warn`. Las variables están al inicio del archivo:

```bash
LEVELS=(20 10 5)    # umbrales en %, de mayor a menor
INTERVAL=30         # segundos entre lecturas
```

Los umbrales deben ir ordenados de mayor a menor. Los avisos con un umbral de 10% o menos se envían con urgencia `critical`.

Después de cualquier cambio, reinicia el servicio:

```bash
systemctl --user restart battery-warn.service
```

## Cómo probarlo

No hace falta descargar la batería. Con el cargador desconectado:

1. Cambia `LEVELS` a valores por encima de tu porcentaje actual. Por ejemplo, si tienes 75%: `LEVELS=(90 80)`.
2. Reinicia el servicio.
3. Espera hasta `INTERVAL` segundos. Debe aparecer la notificación.

Para probar el reinicio de estado: conecta el cargador, espera más de `INTERVAL` segundos, desconéctalo y espera otra lectura. Debe avisar de nuevo. Si conectas y desconectas en menos de `INTERVAL` segundos, el script no llega a leer el estado de carga y no reinicia.

Para ver los valores que lee el script:

```bash
cat /sys/class/power_supply/BAT*/capacity
cat /sys/class/power_supply/BAT*/status
```

Para ver paso a paso qué hace, detén el servicio y ejecútalo en primer plano:

```bash
systemctl --user stop battery-warn.service
bash -x ~/.local/bin/battery-warn
```

## Solución de problemas

**`notify-send: command not found`**
Falta instalar `libnotify`.

**`The name is not activatable`**
No hay ningún demonio de notificaciones instalado o en ejecución. Revisa la sección de requisitos.

**`notify-send` funciona en la terminal pero el servicio no muestra avisos**
El servicio de usuario puede no tener las variables del entorno gráfico. Importa las variables al iniciar la sesión:

```bash
dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY
```

Ejecútalo desde el arranque de tu entorno (por ejemplo, en el mismo bloque de inicio de Hyprland o en `.xinitrc`).

**El servicio aparece como `inactive (dead)`**
Si el equipo no tiene batería, el script termina por diseño. Comprueba con `ls /sys/class/power_supply/`. Si hay una batería pero con un nombre que no empieza por `BAT`, el script no la detecta.

**No avisa aunque el porcentaje está por debajo del umbral**
El script solo avisa cuando el estado es exactamente `Discharging`. Con el cargador conectado, o con el estado `Full` o `Not charging`, no avisa. Además, cada umbral avisa una sola vez por descarga.

**Ver el registro del servicio**

```bash
journalctl --user -u battery-warn.service -n 20
```

## Limitaciones

- Solo vigila la primera batería que encuentra. Equipos con dos baterías no están soportados.
- Usa sondeo: lee la batería cada `INTERVAL` segundos en lugar de reaccionar a eventos.
- Cada umbral avisa una sola vez. Si ignoras el aviso del 20%, no recibes otro hasta llegar al 10%.
- Si la batería baja de un umbral a otro más bajo entre dos lecturas, se omite el aviso intermedio.
- Si el servicio se reinicia, pierde el estado y puede repetir un aviso.
- XFCE incluye `xfce4-power-manager`, que ya muestra avisos de batería baja. Allí este script puede ser redundante.

## Desinstalación

```bash
systemctl --user disable --now battery-warn.service
rm ~/.config/systemd/user/battery-warn.service
rm ~/.local/bin/battery-warn
systemctl --user daemon-reload
```


