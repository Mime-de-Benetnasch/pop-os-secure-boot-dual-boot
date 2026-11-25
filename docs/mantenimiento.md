# Mantenimiento, compatibilidad y recuperación

## Qué hace el gancho y qué no hace

En Pop!_OS con `initramfs-tools`, DKMS prepara los módulos del kernel y `update-initramfs` genera la imagen inicial. El gancho posterior toma **la versión que recibe**, no `uname -r`: al actualizar puede seguir ejecutándose el kernel anterior. Genera una UKI nueva, la firma y verifica primero en un espacio de trabajo privado y luego en una copia temporal en la ESP. Solo tras esas comprobaciones renombra la copia al nombre estable que usa rEFInd. Si un paso falla **antes** del renombrado, el archivo vigente no se sustituye; FAT no ofrece garantías absolutas ante cortes de energía.

La plantilla exige que la ESP sea la esperada por `PARTUUID`, bloquea ejecuciones simultáneas y evita que `update-initramfs -k all` reemplace una UKI válida por un kernel más antiguo. Para cada módulo configurado compara `modinfo -F sig_key` con el serial del certificado DKMS (comportamiento observado en el entorno probado). Esa comparación es una comprobación de metadatos, no valida criptográficamente el módulo ni demuestra que el kernel imponga la política. Verifica además el arranque, los mensajes del kernel y el hardware.

No actualiza `shim`, MokManager, rEFInd ni Windows. No controla la eliminación de kernels o de sus módulos por APT, no detecta por sí solo cambios de parámetros del kernel y no prueba una UKI nueva reiniciando. La plantilla comparte la lógica del gancho usado en el caso, pero tiene una configuración genérica; no es un sustituto directo de un gancho existente con otras rutas.

La plantilla se comprobó en un entorno aislado: construcción y firma de una UKI sintética, conservación de la imagen al recibir un kernel anterior y rechazo ante una ESP incorrecta, un identificador DKMS distinto o un fallo de verificación de la copia. Esa prueba no arrancó la imagen sintética ni sustituye la prueba UEFI en el equipo de destino.

## Instalar y probar el gancho

Hazlo desde la raíz del repositorio, después de crear la entrada firmada directa de [la guía de reproducción](reproduccion.md#4-prueba-una-entrada-directa-y-registra-los-mok). La plantilla usa `/boot/efi`, `/root/pop-secureboot`, `/etc/pop-secureboot` y `EFI/PopSecureBoot`. Si cambiaste alguna ruta, adapta y revisa la plantilla y el menú. Requiere `bash`, `ukify`, `sbverify`, `linux-version`, `python3`, `findmnt`, `flock`, `modinfo` y `openssl` (estas dos últimas para el control DKMS).

1. Confirma que `/etc/pop-secureboot/cmdline` contiene exactamente los parámetros de la entrada de Linux que sabes que funciona. Si cambias los parámetros mediante `kernelstub` más adelante, actualiza también ese archivo y vuelve a generar la UKI. No pongas secretos en la línea del kernel: el contenido de la UKI es legible en la ESP.
2. Confirma con `findmnt` y `lsblk` que `/boot/efi` es la ESP de Pop!_OS. Anota su GUID GPT en `/etc/pop-secureboot/esp-partuuid`, por ejemplo con el bloque siguiente **después de verificar el montaje**:

   ```bash
   ( set -euo pipefail
     esp=/boot/efi
     [[ $(findmnt -rn --mountpoint "$esp" -o FSTYPE) == vfat ]]
     partuuid=$(findmnt -rn --mountpoint "$esp" -o PARTUUID)
     [[ $partuuid =~ ^[[:xdigit:]]{8}(-[[:xdigit:]]{4}){3}-[[:xdigit:]]{12}$ ]]
     printf '%s\n' "$partuuid" |
       sudo install -m 0600 /dev/stdin /etc/pop-secureboot/esp-partuuid
   )
   ```

3. Crea `/etc/pop-secureboot/dkms-modules` como archivo de texto, con `sudoedit`. Escribe los módulos que **realmente** deban estar firmados con `/root/pop-secureboot/dkms.der`, un nombre por línea, sin extensiones. Por ejemplo, para un equipo con esos cuatro módulos de NVIDIA gestionados por DKMS:

   ```text
   nvidia
   nvidia_modeset
   nvidia_uvm
   nvidia_drm
   ```

   Agrega módulos críticos como los de System76 solo si están instalados, son externos y los firma ese mismo certificado. Si no tienes módulos que comprobar, crea un archivo vacío, o con solo un comentario `# Sin módulos DKMS personalizados`. Si tu DKMS identifica las firmas de otra forma, adapta y prueba la comprobación; no ignores un fallo automáticamente.

4. Guarda primero una copia funcional de la UKI en un medio privado. Si vas a instalar la plantilla con Secure Boot activo, prepara también la [ruta anterior independiente](#recuperación). Prueba la ejecución explícita con el kernel que estás usando antes de añadir el gancho permanente:

   ```bash
   k=$(uname -r)
   bash -n examples/zzzz-pop-secureboot &&
     sudo bash examples/zzzz-pop-secureboot "$k" "/boot/initrd.img-$k" &&
     sudo sbverify --cert /root/pop-secureboot/boot.crt \
       /boot/efi/EFI/PopSecureBoot/grubx64.efi
   ```

   Si pasaste por el paso de DKMS, genera el `initramfs` nuevo **antes** de esta prueba. No continúes si falta la firma o si se conservó, por ser más reciente, una imagen distinta de la que querías probar.

5. Si no hay un gancho anterior con ese nombre, instálalo y prueba la integración:

   ```bash
   sudo test ! -e /etc/initramfs/post-update.d/zzzz-pop-secureboot &&
     sudo install -o root -g root -m 0755 examples/zzzz-pop-secureboot \
       /etc/initramfs/post-update.d/zzzz-pop-secureboot &&
     sudo update-initramfs -u -k "$(uname -r)" &&
     sudo sbverify --cert /root/pop-secureboot/boot.crt \
       /boot/efi/EFI/PopSecureBoot/grubx64.efi
   ```

   Espera el mensaje **UKI firmada publicada** y **Signature verification OK**. El aviso `No signature table present` puede referirse al kernel suelto que está empaquetando `ukify`, no a la UKI final; comprueba sobre qué archivo salió. Un error en la **UKI final** sí es motivo para detenerse. Si la verificación falla, no cambies el orden de arranque ni reinicies hasta entender qué archivo quedó en la ESP.

## Después de una actualización

Antes de reiniciar, comprueba que DKMS terminó y que el gancho publicó la versión esperada. Puedes leer `.uname` y verificar la firma sin volcar la línea del kernel:

```bash
uki=/boot/efi/EFI/PopSecureBoot/grubx64.efi
sudo ukify inspect "$uki" --section=.uname:text
sudo sbverify --cert /root/pop-secureboot/boot.crt "$uki"
```

Tras arrancar, compara con `uname -r` y verifica controlador y Secure Boot. Si una actualización de DKMS no regenera el `initramfs`, hazlo explícitamente para la versión correcta cuando DKMS ya haya terminado. Mantén el kernel anterior **y sus módulos en `/lib/modules`** hasta confirmar el nuevo; no hagas una limpieza de kernels justo después de un fallo. Una UKI que se puede cargar puede necesitar todavía esos módulos en disco.

Las copias manuales de `shimx64.efi` y `mmx64.efi` no se mantienen solas. Cuando actualices firmware, `dbx`, SBAT o los certificados de Microsoft, comprueba que Windows y ambos caminos de Pop!_OS siguen funcionando. Si actualizas `shim`, conserva su par de MokManager y prueba primero en una ruta paralela. Si rotas la clave EFI o la de DKMS, registra la nueva MOK antes de depender de ella, firma copias nuevas y prueba ambas rutas antes de retirar la confianza antigua.

## Adaptación a otras placas y equipos

Estas son comprobaciones de portabilidad, no modelos certificados. Consulta el manual de tu revisión de placa y versión de firmware.

| Situación | Qué adaptar o verificar |
| --- | --- |
| ASUS, MSI, Gigabyte, ASRock u OEM | Los nombres, teclas y ubicación de Secure Boot varían. Busca `Secure Boot` y el menú de arranque de una sola vez. En [algunas ASUS](https://www.asus.com/support/faq/1049829/) se usa `Windows UEFI mode`; en otras placas puede ser `Enabled` con modo `Standard`. Comprueba el estado desde el sistema operativo, no solo el texto del menú. |
| Windows arranca pero `shim` no | Una UEFI puede confiar en Windows Boot Manager y no en la CA que firma `shim`. Comprueba la política de CA de terceros, si el fabricante la ofrece, y la compatibilidad con las CA de la generación 2023 y con `dbx`. Si tu firmware no permite un `shim` vigente bajo su política, esta receta no aplica tal cual. |
| Un disco o dos; SATA o NVMe | Usa la ESP real para `--disk` y `--part`, y el `PARTUUID` local para `volume`. En una ESP compartida mantén separados los directorios. En una ESP pequeña mide espacio antes de comprometerte a guardar dos UKI a la vez. |
| `BootNext` se ignora o Windows toma prioridad | Usa el menú UEFI para seleccionar la entrada por nombre o revisa el orden desde el firmware. No sobrescribas `EFI/Microsoft/Boot/bootmgfw.efi` ni la ruta de arranque de emergencia como atajo. |
| Sin NVIDIA, o con otros módulos externos | La cadena EFI es independiente de la GPU. Cambia o deja vacía la lista DKMS según los módulos reales y su origen. No apliques los nombres de módulos o paquetes de otro PC. |
| Cambio de placa, restablecimiento o actualización de UEFI | Las entradas NVRAM y el registro MOK pueden perderse aunque la ESP siga intacta. Vuelve a comprobar las claves del firmware, reconstruye las entradas si hace falta y registra únicamente los certificados propios autorizados. Ten disponible la recuperación de BitLocker, en especial si cambia el TPM. |
| Legacy/CSM, EFI de 32 bits o ARM | Los binarios, rutas y prerrequisitos de esta receta no son trasladables tal cual. No improvises una conversión durante la prueba. |

## Recuperación

1. **No sale rEFInd o Windows no aparece en el menú.** Con Secure Boot todavía activo, prueba desde el menú temporal de la placa: **Pop!_OS Secure Boot directo** para Linux o **Windows Boot Manager** para Windows. Si ambas entradas directas funcionan, revisa la firma y SBAT de rEFInd, la ruta de `shim`, `refind.conf` y el `PARTUUID` de Windows. Si Windows solo acepta el arranque directo con su configuración de BitLocker, conserva esa ruta en vez de desactivar protecciones.
2. **Falla Pop!_OS tanto desde rEFInd como directamente.** Esas dos rutas leen la misma UKI. Una recuperación realmente independiente requiere una UKI anterior firmada que haya arrancado, más su kernel/módulos conservados: puedes guardarla en otra carpeta de la ESP con su propio par `shimx64.efi`/`mmx64.efi` y registrarla como otra entrada; no la incluyas en el destino del gancho. Verifícala y pruébala de antemano con `BootNext`. Una copia fuera de la ESP sirve para restaurar, pero no arranca por sí sola.
3. **No funciona ninguna ruta firmada de Linux.** Usa el medio de rescate previsto y restaura únicamente los archivos que identificaste en el respaldo, sobre la ESP correcta. Si el rescate requiere desactivar Secure Boot temporalmente, restaura solo ese ajuste documentado y no borres las claves UEFI. La ruta antigua sin firmar sirve en ese estado, no como respaldo con Secure Boot activo. Vuelve a activar y comprobar Secure Boot cuando la ruta firmada quede reparada.
4. **BitLocker pide recuperación.** Usa la clave que guardaste en privado y consulta la [guía de Microsoft](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/faq). No la publiques, no borres el TPM y no cambies otras opciones al azar. Si es un equipo administrado, sigue las indicaciones del responsable.
5. **`Security Violation`, `Verification failed` o error SBAT.** Distingue la primera etapa rechazada por la UEFI de la segunda etapa rechazada por `shim`. Verifica MOK desde la propia ruta de `shim`, revisa las firmas y sustituye el componente obsoleto por uno vigente. Que haya una firma no implica que se acepte bajo `dbx` o SBAT. Si `mokutil` indica que la validación de `shim` está desactivada, no des por válida la prueba; corrige la política antes de verificar de nuevo.

Para deshacer solo esta configuración, recupera primero un arranque conocido para cada sistema y restaura su prioridad. Solo después retira, de forma selectiva, el gancho y las entradas que creaste. Comprueba qué archivo usa cada entrada antes de borrar una carpeta: un nombre que diga «prueba» no significa que haya dejado de usarse. No restablezcas de golpe la lista MOK ni elimines archivos de Microsoft, claves del firmware o cargadores de otras distribuciones.
