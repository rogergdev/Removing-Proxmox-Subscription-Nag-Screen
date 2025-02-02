# 🔧 Fix para Eliminar Aviso de Suscripción en Proxmox 8.1.11+

Este repositorio contiene un **script automático** que monitorea y elimina el aviso de suscripción en **Proxmox 8.1.11+**. La solución utiliza `inotify-tools` para detectar cambios en el archivo `proxmoxlib.js` y aplicar el fix automáticamente.

---

## 📌 **Características**
✅ **Automático**: Aplica el fix en cuanto Proxmox actualiza el archivo.  
✅ **Seguro**: No reinicia `pveproxy` innecesariamente.  
✅ **Idempotente**: No hace cambios si el fix ya está aplicado.  
✅ **Persistente**: Se ejecuta al inicio del sistema con `systemd`.  

---

## 📌 **1. Instalación de Dependencias**
Primero, instalamos `inotify-tools`, que nos permitirá detectar cambios en el archivo monitoreado.

```bash
apt update && apt install -y inotify-tools
```

---

## 📌 **2. Crear el Script de Monitoreo**
Creamos el archivo del script:
```bash
nano /root/monitor_fix_proxmox.sh
```

Pegamos el siguiente contenido:

```bash
#!/bin/bash
# Monitorea cambios en proxmoxlib.js y aplica el parche automáticamente sin reiniciar pveproxy innecesariamente.

FILE="/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js"
BACKUP_FILE="${FILE}.bak"
TEMP_FILE="/tmp/proxmoxlib.js.tmp"
LOG_FILE="/var/log/fix_proxmox_subscription.log"
RESTART_COUNT_FILE="/tmp/pveproxy_restart_count"

touch "$LOG_FILE"
touch "$RESTART_COUNT_FILE"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

apply_patch() {
    log_message "Detectado cambio en $FILE. Verificando si es necesario aplicar el parche..."

    if [[ ! -f "$BACKUP_FILE" ]]; then
        cp "$FILE" "$BACKUP_FILE"
        log_message "Respaldo creado en $BACKUP_FILE"
    fi

    sed '/^[[:space:]]*Ext\.Msg\.show({/ s/^\([[:space:]]*\)Ext\.Msg\.show({/\1void ({        \/\/Ext.Msg.show({/' "$FILE" > "$TEMP_FILE"

    if diff -q "$FILE" "$TEMP_FILE" > /dev/null; then
        log_message "✅ El fix ya estaba aplicado. No se realizaron cambios."
        rm -f "$TEMP_FILE"
    else
        mv "$TEMP_FILE" "$FILE"
        log_message "✅ Parche aplicado correctamente."

        CURRENT_DATE=$(date "+%Y-%m-%d")
        LAST_RESTART_DATE=$(cat "$RESTART_COUNT_FILE" 2>/dev/null || echo "")

        if [[ "$LAST_RESTART_DATE" != "$CURRENT_DATE" ]]; then
            echo "$CURRENT_DATE" > "$RESTART_COUNT_FILE"
            systemctl restart pveproxy.service
            log_message "🔄 Reiniciado pveproxy para aplicar los cambios."
        else
            log_message "⏳ pveproxy ya fue reiniciado hoy. No se reiniciará nuevamente."
        fi
    fi
}

log_message "🔍 Iniciando monitoreo de $FILE..."
while true; do
    inotifywait -e modify "$FILE" && apply_patch
    sleep 1  # Pequeña pausa para evitar bucles infinitos
done
```

Guardamos con `CTRL + X`, `Y` y `ENTER`.

Damos permisos de ejecución:
```bash
chmod +x /root/monitor_fix_proxmox.sh
```

---

## 📌 **3. Crear un Servicio systemd**
Para ejecutar el script de forma automática, creamos un servicio en `systemd`.

```bash
nano /etc/systemd/system/monitor_fix_proxmox.service
```

Pegamos el siguiente contenido:

```ini
[Unit]
Description=Monitorea cambios en proxmoxlib.js y aplica el parche automáticamente
After=network.target

[Service]
ExecStart=/root/monitor_fix_proxmox.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Guardamos y cerramos el archivo (`CTRL + X`, `Y`, `ENTER`).

Recargamos `systemd`, habilitamos y arrancamos el servicio:
```bash
systemctl daemon-reload
systemctl enable --now monitor_fix_proxmox.service
```

---

## 📌 **4. Restaurar el Backup (Deshacer Cambios)**
Si deseas restaurar la versión original del archivo modificado:
```bash
cp /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js.bak /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy.service
```
Esto restaurará el archivo original y aplicará los cambios en el sistema.

---

## 📌 **5. Desinstalar el Fix y el Servicio**
Para eliminar completamente el fix y su monitoreo:

```bash
systemctl stop monitor_fix_proxmox.service
systemctl disable monitor_fix_proxmox.service
rm /etc/systemd/system/monitor_fix_proxmox.service
rm /root/monitor_fix_proxmox.sh
rm /var/log/fix_proxmox_subscription.log
systemctl daemon-reload
```
Esto eliminará el servicio, el script y los logs del sistema.

---

## 🎯 **Conclusión**
Con este método:
- **El fix se aplica automáticamente** cuando Proxmox actualiza el archivo.
- **No reinicia `pveproxy` innecesariamente**, evitando problemas en el sistema.
- **Es idempotente** (no modifica el archivo si ya está parcheado).
- **Se ejecuta en segundo plano con `systemd`**.

✅ **¡Ahora puedes olvidarte del aviso de suscripción en Proxmox!** 🚀

