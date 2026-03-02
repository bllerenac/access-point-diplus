# Configuración de create_ap en Dispositivo Remoto

Este documento detalla los pasos realizados para transferir y configurar el punto de acceso WiFi en el dispositivo `192.168.137.123`.

## 1. Transferencia de Archivos
Debido a restricciones en el protocolo SFTP del dispositivo, se utilizó el protocolo original de SCP (`-O`).

**Comando ejecutado desde Windows:**
```powershell
scp -O -r "c:\Users\Intel\Desktop\create_ap\create_ap" root@192.168.137.123:/mnt/storage/
```

## 2. Preparación en el Dispositivo
1. Entrar por SSH: `ssh root@192.168.137.123`
2. Dar permisos de ejecución:
   ```bash
   chmod +x /mnt/storage/create_ap/create_ap
   ```

## 3. Configuración del Servicio (Auto-arranque)
Se creó un servicio de `systemd` para que el WiFi inicie solo:

```bash
tee /etc/systemd/system/create_ap.service << 'EOF'
[Unit]
Description=WiFi Access Point
After=network.target
[Service]
Type=simple
ExecStartPre=/bin/killall wpa_supplicant
ExecStartPre=/mnt/storage/create_ap/create_ap --stop wlan0
ExecStartPre=/bin/rm -rf /tmp/create_ap.*
ExecStartPre=/bin/sleep 2
# Cambiamos eth0 por rmnet_ipa0 (tus datos móviles)
ExecStart=/mnt/storage/create_ap/create_ap --no-virt wlan0 rmnet_ipa0 WLGT04 '#%marcobre%#'
ExecStop=/mnt/storage/create_ap/create_ap --stop wlan0
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

## 4. Comandos de Gestión
Ejecutar estos comandos en el dispositivo para activar todo:

- **Recargar configuración:** `systemctl daemon-reload`
- **Activar al arrancar:** `systemctl enable create_ap`
- **Iniciar ahora:** `systemctl start create_ap`
- **Ver estado/errores:** `systemctl status create_ap`
- **Ver logs en tiempo real:** `journalctl -u create_ap -f`
