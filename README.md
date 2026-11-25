# Pop!_OS y Windows 11 en arranque dual con Secure Boot

Pop!_OS 24.04 LTS y Windows 11 arrancaron desde el mismo menú de **rEFInd** con **UEFI Secure Boot activo** en ambos. No hizo falta reinstalar, reparticionar, reemplazar las claves de fábrica ni modificar Windows Boot Manager. Esta guía reconstruye lo que funcionó y muestra cómo adaptarlo a otro PC x86-64 sin copiar la configuración del equipo original.

> **No es una configuración oficial de Pop!_OS ni un instalador.** [System76 indica que Secure Boot debe estar desactivado para instalar Pop!_OS](https://system76.com/support/install-pop/#secure-boot). Aquí se parte de dos sistemas ya instalados y arrancables en UEFI. Antes de cambiar el arranque: guarda un respaldo, comprueba que puedes usar un medio de rescate y ten a mano la clave de recuperación de BitLocker si aplica.

- [Reproducción paso a paso](docs/reproduccion.md): comprobar la UEFI y las particiones, preparar las firmas, probar el arranque y configurar rEFInd.
- [Mantenimiento y recuperación](docs/mantenimiento.md): actualizaciones del kernel, problemas comunes, cambios de placa y vuelta a una configuración funcional.
- [Plantilla del gancho de actualización](examples/zzzz-pop-secureboot): versión adaptable y sin valores del equipo original.

## Cómo queda el arranque

```text
UEFI con Secure Boot activo y claves del fabricante/Microsoft intactas
├─ Entrada principal → shim firmado → rEFInd firmado con MOK local
│  ├─ Pop!_OS   → UKI firmada con MOK local → kernel + initramfs + parámetros
│  └─ Windows 11 → Windows Boot Manager original, firmado por Microsoft
├─ Entrada de respaldo → shim firmado → la misma UKI de Pop!_OS
└─ Entrada directa de Windows → Windows Boot Manager original
```

La UEFI verifica `shim` contra sus propias bases de confianza. `shim` permite usar certificados locales registrados con **MokManager** sin tener que añadirlos a `db` ni cambiar `PK`, `KEK` o `dbx`. La **UKI** reúne el kernel, el `initramfs` y los parámetros del kernel en un solo ejecutable EFI firmado. Los módulos externos preparados con **DKMS** llevan una firma separada; no quedan autorizados para el kernel por el solo hecho de meterlos en la UKI.

## Qué hicimos y por qué

| Etapa | Qué pasó |
| --- | --- |
| Menú inicial | Pop!_OS y Windows ya tenían entradas UEFI y particiones EFI separadas. Primero se configuró rEFInd como selector gráfico, con Pop!_OS por defecto y una espera que quedó en 10 segundos. |
| Diagnóstico | `sbverify` mostró que el rEFInd anterior, `systemd-boot` y el kernel de esa ruta no tenían firma. Los módulos externos de NVIDIA tampoco mostraban firma. Activar Secure Boot sin preparar otra ruta habría bloqueado ese arranque. |
| Primera prueba | Se verificó el respaldo EFI y se extrajo de APT el `shim` firmado con doble firma de Ubuntu, junto con su MokManager, sin instalar el paquete ni tocar el cargador anterior. Se preparó una UKI inicial y se comprobó la ruta directa con `BootNext`, todavía sin activar Secure Boot. |
| Registro de confianza | Se registraron dos certificados MOK: uno para ejecutables EFI y otro restringido a módulos. Después de confirmar el registro, un reinicio volvió a la ruta anterior; desde ahí no se veían los MOK. Arrancar de nuevo por `shim` permitió confirmar ambos sin repetir el registro. |
| Controladores y prueba real | DKMS reconstruyó y firmó los módulos externos de NVIDIA y System76. Se regeneró el `initramfs` y se volvió a firmar la UKI. Tras activar Secure Boot sin borrar claves de la UEFI, Pop!_OS arrancó por la entrada directa y `nvidia-smi` reconoció la GPU. |
| Menú definitivo | El rEFInd viejo no tenía `.sbat`. Se verificó una copia de rEFInd 0.14.2 con metadatos SBAT, se firmó con la clave local y se instaló junto a una segunda copia de `shim`. Se probaron las dos entradas del menú antes de hacerlo predeterminado. La de Linux apuntó a la UKI, no al `systemd-boot` sin firmar; la de Windows conservó su cargador original en su propia ESP. |
| Actualizaciones | Se instaló un gancho posterior a `initramfs-tools` que verifica los módulos NVIDIA esperados, reconstruye y firma la UKI, y verifica la copia antes de sustituir el archivo al que apuntan rEFInd y la entrada directa. Se comprobó la reconstrucción y la firma con el kernel que estaba en uso. |

**Verificación del caso:** en Pop!_OS se observó `SecureBoot enabled`, la entrada UEFI esperada como `BootCurrent` y el controlador NVIDIA funcionando. En Windows 11 se confirmó **Estado de arranque seguro: Activado** en `msinfo32` después de elegirlo en rEFInd. La sesión también mostró que el kernel de Pop!_OS no había activado automáticamente `lockdown`: el estado de Secure Boot de la UEFI, la validación de la cadena EFI y la política con la que el kernel acepta módulos son comprobaciones distintas. El procedimiento no demuestra compatibilidad con todas las placas ni con cada actualización futura.

El tema del menú era cosmético. Se conservaron el fondo y el selector; el icono genérico de Windows podía sustituirse indicando `icon` antes de `volume`, y el espaciado se podía ajustar con PNG con margen transparente. No hacía falta volver a firmar rEFInd para cambiar esos recursos. Este repositorio no incluye ese tema, capturas, logs, claves, certificados locales, binarios EFI ni identificadores de la instalación.

## Referencias

- System76: [instalación de Pop!_OS](https://system76.com/support/install-pop/#secure-boot), [arranque dual con Windows](https://system76.com/support/windows/#dual-booting-windows) y [reparación del gestor de arranque](https://system76.com/support/bootloader/).
- Microsoft: [Secure Boot y sus bases de confianza](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-secure-boot), [transición de certificados de Secure Boot](https://aka.ms/GetSecureBoot) y [recuperación de BitLocker](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/faq).
- `shim`: [proyecto](https://github.com/rhboot/shim), [SBAT y revocaciones](https://github.com/rhboot/shim/blob/main/SBAT.md), [paquete firmado de Ubuntu Noble](https://launchpad.net/ubuntu/noble/+package/shim-signed).
- rEFInd: [Secure Boot](https://www.rodsbooks.com/refind/secureboot.html), [descargas oficiales](https://www.rodsbooks.com/refind/getting.html) y [configuración del menú](https://www.rodsbooks.com/refind/configfile.html).
- UKI y módulos: [especificación de UKI](https://uapi-group.org/specifications/specs/unified_kernel_image/), [`ukify`](https://www.freedesktop.org/software/systemd/man/latest/ukify.html), [DKMS](https://github.com/dell/dkms) y [firma de módulos del kernel](https://docs.kernel.org/admin-guide/module-signing.html).
