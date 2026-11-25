# Reproducir el arranque dual en otro equipo

Parte de Pop!_OS y Windows 11 funcionales en UEFI x86-64. Realiza las pruebas en persona: necesitas acceso al menú de arranque y a MokManager. No ejecutes toda la guía como un solo script. Las variables de Bash se pierden al reiniciar: vuelve a definirlas y consulta las entradas reales antes de cada paso.

## 1. Inventario y respaldo

En Pop!_OS, comprueba el modo de arranque, dónde está montada su partición EFI (ESP), cuánto espacio tiene y qué entradas existen:

```bash
test -d /sys/firmware/efi && cat /sys/firmware/efi/fw_platform_size
sudo bootctl --no-pager status
sudo efibootmgr -v
findmnt --mountpoint /boot/efi -o TARGET,SOURCE,FSTYPE,PARTUUID
lsblk -o PATH,PKNAME,PARTN,PARTTYPE,PARTUUID,FSTYPE,MOUNTPOINTS
df -h /boot/efi
mokutil --sb-state
```

- `fw_platform_size` debería mostrar `64`, y `uname -m`, `x86_64`. Si algún sistema arranca en modo BIOS/CSM, detente: convertirlo a UEFI es otro procedimiento. En Windows, `msinfo32` debe indicar **Modo de BIOS: UEFI**.
- Relaciona la entrada **Windows Boot Manager** de `efibootmgr -v` con su ESP mediante el GUID de `HD(…,GPT,…)`. Para rEFInd interesa el **`PARTUUID`** GPT, no el `UUID` del sistema de archivos ni el de la raíz de Linux. Si están en discos distintos, no hace falta copiar `EFI/Microsoft` a la ESP de Pop!_OS. Si comparten ESP, tampoco se mueve ni se sobrescribe esa carpeta.
- Comprueba el espacio para la UKI vigente **más** una UKI temporal durante cada publicación. Si no cabe, detente: no borres cargadores, no formatees y no redimensiones aquí.
- Guarda la clave de recuperación de BitLocker o del cifrado de dispositivo, si se usa, fuera del PC. Los cambios de UEFI, orden de arranque o cargador pueden provocar una solicitud de recuperación. Si el equipo está administrado o no puedes recuperar el volumen, no sigas sin consultar al responsable.

Ejemplo de respaldo cuando Pop!_OS tiene su ESP montada en `/boot/efi`:

```bash
esp=/boot/efi
( umask 077
  test "$(findmnt -rn --mountpoint "$esp" -o FSTYPE)" = vfat &&
    backup=$(mktemp -d "$HOME/respaldo-efi.XXXXXX") &&
    sudo tar -C "$esp" -czf - . > "$backup/pop-esp.tar.gz" &&
    sudo efibootmgr -v > "$backup/entradas-uefi.txt" &&
    tar -tzf "$backup/pop-esp.tar.gz" >/dev/null &&
    printf 'Respaldo legible en %s\n' "$backup"
)
```

Guarda una copia fuera del equipo. Si Windows tiene otra ESP, respáldala por separado y no la montes en `/boot/efi` encima de la de Pop!_OS. El archivo de `efibootmgr` sirve como referencia para reconstruir las entradas, no es un volcado restaurable de la NVRAM. Ten también un medio de rescate probado. Si un medio de Pop!_OS no es aceptado con Secure Boot activo, planifica cómo recuperar el acceso antes de empezar, sin borrar las claves del firmware.

## 2. Consigue un `shim` compatible, sin instalar otro gestor

Instala las herramientas según los paquetes disponibles en tu versión de Pop!_OS. Revisa lo que propone APT antes de confirmar:

```bash
sudo apt install --no-install-recommends systemd-ukify systemd-boot-efi \
  sbsigntool mokutil efibootmgr openssl binutils initramfs-tools linux-base
```

Descarga `shim-signed` desde los repositorios de confianza ya configurados, **sin instalarlo**. Así no ejecutas sus scripts de instalación ni agregas otra cadena de arranque. Usa un directorio nuevo y conserva su ruta para el paso 4:

```bash
work=$(mktemp -d "$HOME/preparar-secureboot.XXXXXX")
( set -e
  cd "$work"
  apt-cache policy shim-signed
  apt-get download shim-signed
  dpkg-deb -x ./shim-signed_*.deb paquete
  find paquete/usr/lib/shim -maxdepth 1 -type f -name '*.efi*' -print
)
```

En el caso documentado funcionó Ubuntu `shim` 15.8, variante `shimx64.efi.dualsigned`, con `mmx64.efi` del mismo paquete. **Es un dato histórico, no una versión que debas fijar.** Elige un `shim` vigente y que la `db` de tu UEFI acepte. En algunas versiones los archivos cambian de nombre o MokManager se distribuye en otro paquete de la misma compilación. No mezcles versiones ni vuelvas a una versión revocada.

```bash
# Ajusta el nombre del archivo según el paquete; no asumas que todos son iguales.
shim="$work/paquete/usr/lib/shim/shimx64.efi.dualsigned"
mm="$work/paquete/usr/lib/shim/mmx64.efi"
sbverify --list "$shim"
sbverify --list "$mm"
sudo mokutil --db
sudo mokutil --list-sbat-revocations
```

`sbverify --list` enumera firmas, no verifica que el firmware acepte el archivo. Para comprobar una cadena concreta fuera del arranque, usa `sbverify --cert` con un certificado autenticado de la cadena correspondiente; la prueba de aceptación final es arrancar con Secure Boot activo. No confundas la CA que permite arrancar Windows con la CA de terceros que puede necesitar `shim`. Revisa la transición de certificados de Microsoft, `dbx` y SBAT; no desactives sus comprobaciones para que un binario arranque.

## 3. Prepara la UKI y la clave local de arranque

Usa los parámetros de la entrada **funcional** de Pop!_OS. En instalaciones con systemd-boot suelen estar en la línea `options` de `/boot/efi/loader/entries/Pop_OS-current.conf`. Si el archivo no coincide con la instalación o usas cifrado, LVM, Btrfs u otra raíz, verifica la entrada real y conserva todos sus parámetros necesarios. No copies los UUID de otra máquina, ni añadas parámetros de rescate que se usaron una sola vez. Si hay varios `initrd` (por ejemplo, microcódigo separado), incorpora todos en el orden original mediante opciones `--initrd` sucesivas o confirma que el `initramfs` local ya los contiene; la plantilla usa un solo `initramfs` completo.

Para una **instalación nueva de esta configuración**, ejecuta el bloque. Si ya existe `/root/pop-secureboot` o `/etc/pop-secureboot`, se detiene para no sobrescribir claves o parámetros:

```bash
keys=/root/pop-secureboot
cfg=/etc/pop-secureboot
esp=/boot/efi
k=$(uname -r)

( set -euo pipefail
  sudo test ! -e "$cfg"
  sudo mkdir -m 0700 "$keys"
  sudo mkdir -m 0700 "$cfg"
  sudo sed -nE 's/^[[:space:]]*options[[:space:]]+//p' \
    "$esp/loader/entries/Pop_OS-current.conf" |
    sudo install -m 0600 /dev/stdin "$cfg/cmdline"
  sudo awk 'NF {n++} END {exit (n != 1)}' "$cfg/cmdline"
  sudo openssl req -new -x509 -newkey rsa:2048 -noenc -days 3650 \
    -subj '/CN=Local EFI signing/' \
    -addext 'basicConstraints=critical,CA:FALSE' \
    -addext 'keyUsage=critical,digitalSignature' \
    -addext 'extendedKeyUsage=codeSigning' \
    -keyout "$keys/boot.key" -out "$keys/boot.crt"
  sudo chmod 0600 "$keys/boot.key"
  sudo openssl x509 -in "$keys/boot.crt" -outform DER -out "$keys/boot.der"
  sudo test -s "/boot/initrd.img-$k"
  sudo ukify --config=/dev/null build --linux="/boot/vmlinuz-$k" \
    --initrd="/boot/initrd.img-$k" --cmdline="@$cfg/cmdline" \
    --uname="$k" --os-release=@/etc/os-release \
    --secureboot-private-key="$keys/boot.key" \
    --secureboot-certificate="$keys/boot.crt" --output="$keys/popos.efi"
  sudo sbverify --cert "$keys/boot.crt" "$keys/popos.efi"
)
```

Las claves privadas quedan sin contraseña para que los ganchos puedan firmar automáticamente. Esto implica confiar en `root`: cualquiera que obtenga esa clave puede firmar código para los equipos que la hayan registrado. Mantén las claves fuera de la ESP y del repositorio, protegidas con permisos y con un respaldo cifrado. Genera claves distintas por equipo.

## 4. Prueba una entrada directa y registra los MOK

Identifica el disco completo (`disk`) y el número de partición (`part`) correspondientes **a la ESP de Pop!_OS** en el inventario. No uses el dispositivo de la partición como argumento de `--disk`. Aquí y en el paso de rEFInd, si `mkdir` indica que el destino ya existe, detente y revisa su contenido en lugar de sobrescribirlo.

```bash
read -r -p 'Ruta del disco completo de la ESP de Pop!_OS: ' disk
read -r -p 'Número de la partición EFI de Pop!_OS: ' part

( set -euo pipefail
  [[ -b $disk && $part =~ ^[1-9][0-9]*$ ]]
  [[ $(lsblk -dn -o TYPE "$disk") == disk ]]
  esp_dev=$(findmnt -rn --mountpoint "$esp" -o SOURCE)
  [[ $(findmnt -rn --mountpoint "$esp" -o FSTYPE) == vfat ]]
  [[ $(lsblk -dn -o PKNAME "$esp_dev") == "$(lsblk -dn -o KNAME "$disk")" ]]
  [[ $(lsblk -dn -o PARTN "$esp_dev") == "$part" ]]
  sudo mkdir "$esp/EFI/PopSecureBoot"
  sudo cp -- "$shim" "$esp/EFI/PopSecureBoot/shimx64.efi"
  sudo cp -- "$mm" "$esp/EFI/PopSecureBoot/mmx64.efi"
  sudo cp -- "$keys/popos.efi" "$esp/EFI/PopSecureBoot/grubx64.efi"
  sudo sbverify --cert "$keys/boot.crt" "$esp/EFI/PopSecureBoot/grubx64.efi"
  sudo sync -f "$esp"
  sudo efibootmgr --create-only --disk "$disk" --part "$part" \
    --label 'Pop!_OS Secure Boot directo' --loader '\EFI\PopSecureBoot\shimx64.efi'
)
```

`grubx64.efi` es el nombre de segunda etapa que espera ese `shim` x86-64; el archivo en esta carpeta es una **UKI**, no GRUB. Confirma el nombre esperado en la documentación de la variante de `shim` que estés usando. No registres la UKI directamente en la UEFI si su certificado solo está en el MOK de `shim`.

Con `sudo efibootmgr`, anota el identificador hexadecimal de cuatro caracteres de la nueva entrada. Para probar una vez, ejecuta `sudo efibootmgr --bootnext "$id"` tras asignar ese identificador real a `id`. Verifica `BootNext` y que el orden previo se conserve (algunos firmwares añaden la entrada al final). Si Pop!_OS estaba arrancando con Secure Boot desactivado, haz una primera prueba en ese estado y verifica que `BootCurrent` es la entrada nueva. Esto demuestra que la ruta funciona, **no** que las firmas se acepten todavía. Si ya estabas usando una ruta con Secure Boot activo, no lo desactives para esta prueba: registra el MOK y usa la nueva entrada con Secure Boot activo.

Si tienes módulos externos gestionados por DKMS, prepara también su clave, distinta de la de arranque. El OID `1.3.6.1.4.1.2312.16.1.2` marca ese certificado para firma de módulos: no lo uses para firmar la UKI o rEFInd.

```bash
keys=/root/pop-secureboot
sudo openssl req -new -x509 -newkey rsa:2048 -noenc -days 3650 \
  -subj '/CN=Local DKMS module signing/' \
  -addext 'basicConstraints=critical,CA:FALSE' \
  -addext 'keyUsage=critical,digitalSignature' \
  -addext 'extendedKeyUsage=codeSigning,1.3.6.1.4.1.2312.16.1.2' \
  -keyout "$keys/dkms.key" -outform DER -out "$keys/dkms.der" &&
  sudo chmod 0600 "$keys/dkms.key"
```

No repitas la generación si ya existen esos archivos. Solicita el registro con `sudo mokutil --import "$keys/boot.der" "$keys/dkms.der"`; sin DKMS, importa solo `boot.der`. Escribe la contraseña temporal únicamente en la terminal y en **MokManager → Enroll MOK → Continue → Yes**. Programa la entrada directa con `--bootnext` para que ejecute MokManager al reiniciar. Tras confirmar y reiniciar, vuelve a seleccionar la entrada de `shim` si `BootNext` ya se consumió. Comprueba `BootCurrent` y `sudo mokutil --test-key "$keys/boot.der"`; repite `--test-key` con `dkms.der` si corresponde. Si arrancaste por la ruta antigua, cambia de entrada antes de concluir que el registro falló.

## 5. Firma DKMS antes de regenerar el initramfs

Omite esta sección si no utilizas módulos externos que deban firmarse con DKMS. Para otros equipos, no asumas que tienen los mismos módulos que el caso original: revisa `dkms status`, identifica los módulos críticos (GPU, red, almacenamiento, etc.) y comprueba que existen los encabezados de la versión `uname -r`. Guarda un respaldo local de los módulos actuales antes de recompilar.

Crea con `sudoedit` un archivo `/etc/dkms/framework.conf.d/90-pop-secureboot.conf` con el siguiente contenido. Revisa si ya existe otra configuración de firma; el cambio afecta a los módulos que ese DKMS reconstruya, no solo a NVIDIA.

```sh
mok_signing_key="/root/pop-secureboot/dkms.key"
mok_certificate="/root/pop-secureboot/dkms.der"
```

Para **cada** `módulo/versión` que identifiques como instalado por DKMS, introduce el nombre informado por `dkms status`, sin la coma ni el estado:

```bash
k=$(uname -r)
read -r -p 'Módulo/versión DKMS a recompilar: ' dkms_pkg
sudo dkms build "$dkms_pkg" -k "$k" --force &&
  sudo dkms install "$dkms_pkg" -k "$k" --force
```

`--force` puede reemplazar los módulos instalados. Si falla, no reinicies: revisa el log de DKMS, los encabezados y la compatibilidad del controlador con ese kernel. Comprueba con `modinfo -k "$k" -F signer nombre_del_modulo` que los módulos muestran tu certificado. Algunos equipos usan módulos precompilados firmados por su distribución en vez de módulos DKMS; no los recompiles ni obligues al gancho a verificarlos con tu clave si no es la que los firma.

Después de verificar, ejecuta `sudo update-initramfs -u -k "$k"`, reconstruye la UKI **desde ese nuevo initramfs** y verifica la imagen antes de sustituirla. Para esta reconstrucción puedes usar la plantilla tal como describe [Mantenimiento: instalar el gancho](mantenimiento.md#instalar-y-probar-el-gancho), dejando la imagen anterior intacta si falla el paso de firma. Sin DKMS, la primera UKI ya puede servir para la prueba, siempre que corresponda al `initramfs` correcto.

## 6. Activa Secure Boot y valida Linux por la entrada directa

Activa Secure Boot en la UEFI manteniendo sus claves. Según el firmware puede llamarse **Secure Boot Enabled**, o en algunas ASUS **Boot → Secure Boot → OS Type → Windows UEFI mode**. Ese ajuste no impide arrancar Linux firmado. No uses **Clear Secure Boot Keys**, no vacíes el TPM y no cambies a `Custom` o a `Setup Mode` para este método. Si el firmware tiene un selector oficial para permitir la CA de terceros de Microsoft, revisa su manual y la política del equipo antes de usarlo.

Desde el menú de arranque de la placa, elige **Pop!_OS Secure Boot directo**. En Linux, comprueba:

```bash
mokutil --sb-state
sudo efibootmgr | grep '^BootCurrent:'
uname -r
nvidia-smi -L   # solo si usas NVIDIA; no compartas los UUID que muestra
cat /sys/kernel/security/lockdown  # si el archivo existe
```

Busca `SecureBoot enabled`, sin advertencia de validación desactivada en `shim`, la entrada correcta, el kernel esperado y el hardware funcional. Secure Boot activo no implica que `lockdown` esté en `integrity` o `confidentiality`. Si la entrada falla, usa [recuperación](mantenimiento.md#recuperación) antes de cambiar el orden.

## 7. Firma rEFInd y añade Windows al menú

Usa [una distribución oficial de rEFInd](https://www.rodsbooks.com/refind/getting.html) para x86-64 que tenga una sección `.sbat` aceptada por tu `shim`. Comprueba origen e integridad con los mecanismos de la distribución o del proyecto. Si partes del ZIP firmado por el autor, verifica el binario con su certificado obtenido de una fuente confiable, no solo con un certificado adjunto a una descarga de origen desconocido. En el caso documentado se verificó el binario de rEFInd 0.14.2, se trabajó sobre una copia, se quitó su firma previa con `sbattach --remove` y se firmó con el MOK local. **No edites ni regeneres `.sbat` para eludir una revocación.**

En el siguiente bloque, `refind_dir` debe apuntar al directorio verificado que contiene `refind_x64.efi` e `icons/` (dentro de la distribución de rEFInd). Vuelve a confirmar `esp`, `disk` y `part` como en el paso 4. `-d` de `mktemp` crea un directorio nuevo; se firma la copia, nunca el binario original:

```bash
read -r -p 'Directorio verificado con el binario y los iconos de rEFInd: ' refind_dir
( set -euo pipefail
  test "$(findmnt -rn --mountpoint "$esp" -o FSTYPE)" = vfat
  dir="$esp/EFI/RefindSecureBoot"
  src=$(mktemp -d)
  cp -- "$refind_dir/refind_x64.efi" "$src/refind.efi"
  objdump -s -j .sbat "$src/refind.efi"
  # Para una distribución verificada que ya venía firmada, como el ZIP oficial:
  sbattach --remove "$src/refind.efi"
  objdump -s -j .sbat "$src/refind.efi"
  sudo sbsign --key "$keys/boot.key" --cert "$keys/boot.crt" \
    --output "$src/grubx64.efi" "$src/refind.efi"
  sudo sbverify --cert "$keys/boot.crt" "$src/grubx64.efi"
  sudo mkdir "$dir"
  sudo cp -- "$src/grubx64.efi" "$esp/EFI/PopSecureBoot/"{shimx64,mmx64}.efi "$dir/"
  sudo cp -r -- "$refind_dir/icons" "$dir/"
  sudo sbverify --cert "$keys/boot.crt" "$dir/grubx64.efi"
)
```

Si la fuente autenticada ya trae un binario **sin firmar**, omite `sbattach --remove` y conserva la comprobación de `.sbat`. No cargues controladores EFI de rEFInd innecesarios: los dos ejecutables se leen de particiones FAT.

Crea `refind.conf` con `sudoedit /boot/efi/EFI/RefindSecureBoot/refind.conf`. Sustituye `PARTUUID_ESP_WINDOWS` por el GUID GPT de la ESP de Windows o quita `volume` si el archivo está en la misma ESP que rEFInd. Verifica la existencia real de `bootmgfw.efi`. Las rutas son relativas a la raíz del volumen EFI, no al sistema de archivos de Linux. No añadas `options` ni `initrd` a la entrada de Pop!_OS: van dentro de la UKI.

```text
timeout 10
default_selection "Pop!_OS"
scanfor manual
use_nvram false

menuentry "Pop!_OS" {
    icon /EFI/RefindSecureBoot/icons/os_linux.png
    loader /EFI/PopSecureBoot/grubx64.efi
}

menuentry "Windows 11" {
    icon /EFI/RefindSecureBoot/icons/os_win8.png
    volume PARTUUID_ESP_WINDOWS
    loader /EFI/Microsoft/Boot/bootmgfw.efi
}
```

Si el paquete no trae esos iconos, elige nombres existentes u omite `icon`. Cuando la ESP de Windows es distinta, colocar `icon` **antes** de `volume` permite usar el icono de la ESP de rEFInd. Puedes añadir un tema con `include`, `banner` y `selection_big`, o ajustar `big_icon_size` sin volver a firmar el ejecutable.

```bash
sudo sync -f "$esp" &&
  sudo efibootmgr --create-only --disk "$disk" --part "$part" \
    --label 'rEFInd Secure Boot' --loader '\EFI\RefindSecureBoot\shimx64.efi'
```

1. Usa el identificador de esta nueva entrada con `--bootnext` y prueba **Pop!_OS desde rEFInd**. Comprueba otra vez Secure Boot, `BootCurrent` y el controlador de vídeo.
2. Programa otra prueba y elige **Windows 11 desde rEFInd**. En Windows, abre `msinfo32` y verifica **Modo de BIOS: UEFI** y **Estado de arranque seguro: Activado**. PowerShell en modo administrador ofrece `Confirm-SecureBootUEFI` como verificación adicional; debe devolver `True`. Si aparece recuperación de BitLocker, no borres el TPM; consulta [recuperación](mantenimiento.md#recuperación).
3. Solo después de comprobar ambos, pon **rEFInd Secure Boot** primero desde el menú UEFI o con `efibootmgr --bootorder` y una lista de identificadores que hayas leído **en ese equipo**. Conserva tanto la entrada directa firmada de Pop!_OS como Windows Boot Manager. Reinicia sin `BootNext` y comprueba que aparece el menú.

La entrada directa de Linux evita depender del menú de rEFInd, pero comparte la misma UKI. Para sobrevivir a una UKI defectuosa, prepara también una copia anterior conocida que arranque, como describe la [guía de mantenimiento](mantenimiento.md).
