# Configuración de create_ap en Modo Repetidor WiFi (AP + STA)

Este documento detalla los pasos para configurar tu dispositivo como un repetidor WiFi. En este modo, el dispositivo utilizará su **única antena interna (`wlan0`)** para conectarse a una red WiFi existente (para tener internet) y al mismo tiempo emitir una nueva red WiFi virtual.

## Requisitos Previos

1.  **Conexión a Internet:** Antes de iniciar el servicio `create_ap`, tu dispositivo **YA DEBE ESTAR CONECTADO** a la red WiFi que proveerá el internet (por ejemplo, a la red `UNDIS_FMS_CL`).
    Esto generalmente lo maneja `wpa_supplicant` o `NetworkManager` en segundo plano. No debemos detener ese servicio.
2.  Tener el script `create_ap` ya transferido al dispositivo en `/mnt/storage/create_ap/` con permisos de ejecución.

---

## 1. Configuración del Servicio de Auto-arranque (systemd)

Entra por SSH a tu dispositivo (`ssh root@192.168.137.123`) y ejecuta el siguiente bloque de comandos para crear o actualizar el archivo de servicio.

**Copia y pega todo este bloque en la terminal de tu dispositivo:**

```bash
tee /etc/systemd/system/create_ap.service << 'EOF'
[Unit]
Description=WiFi Access Point (Repeater Mode)
After=network.target wifi-autoconnect.service

[Service]
Type=simple

# Detenemos cualquier instancia anterior de create_ap
ExecStartPre=/mnt/storage/www/access-point-diplus/create_ap --stop wlan0
ExecStartPre=/bin/rm -rf /tmp/create_ap.*
ExecStartPre=/bin/sleep 2

# IMPORTANTE: NO hacemos killall a wpa_supplicant para no perder la conexión a internet.

# Iniciamos create_ap en Modo Concurrente (AP+STA) usando la misma interfaz wlan0
# Sintaxis: create_ap <interface_ap_virtual> <interface_con_internet> <SSID> <Password>
ExecStart=/mnt/storage/www/access-point-diplus/create_ap wlan0 wlan0 WLANVT31 '12345678'

ExecStop=/mnt/storage/www/access-point-diplus/create_ap --stop wlan0
ExecStopPost=/bin/rm -rf /tmp/create_ap.*
Restart=on-failure
RestartSec=10s
User=root
WorkingDirectory=/
Environment="HOME=/"

[Install]
WantedBy=multi-user.target
EOF
```

---

## 2. Aplicar los Cambios y Arrancar

Una vez creado/modificado el archivo, ejecuta estos comandos en el dispositivo para que el sistema tome los cambios y arranque la red virtual:

```bash
# 1. Recargar los archivos de configuración de systemd
systemctl daemon-reload

# 2. Habilitar el servicio para que inicie al encender el equipo (si no estaba ya habilitado)
systemctl enable create_ap

# 3. Iniciar el servicio ahora mismo
systemctl start create_ap
```

---

## 3. Verificación y Monitoreo

Para comprobar que todo está funcionando correctamente, puedes usar estos comandos:

- **Ver el estado del servicio:**
  ```bash
  systemctl status create_ap
  ```
  *(Debería decir "Active: active (running)" y abajo mostrar que la interfaz virtual `ap0` o similar ha sido creada)*

- **Ver los logs en tiempo real (si hay problemas para conectarse o se cae):**
  ```bash
  journalctl -u create_ap -f
  ```

- **Verificar las interfaces de red actuales:**
  ```bash
  iw dev
  ```
  *(Deberías ver tu `wlan0` conectada a tu red principal, y una nueva interfaz -probablemente llamada `ap0`- transmitiendo el SSID `WLGT04`)*
