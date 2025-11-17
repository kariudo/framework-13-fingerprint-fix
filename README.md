# framework-13-fingerprint-fix
Fix for Framework 13 AMD fingerprint sensor not working sometimes following wake.

This will check for the Goodix device on the bus on wake, and if it is not seen, restart the appropriate PCI function to reload the devices on the controller:

`Bus 001 Device 002: ID 27c6:609c Shenzhen Goodix Technology Co.,Ltd. Goodix Fingerprint USB Device`

## Instructions

### Create the script 

```sh
> sudo mkdir -p /etc/local/bin
> sudo vim run-after-wake-for-fprint.sh
```

Fill it with the following:

```sh
#!/bin/sh
# Rebind USB controller 0000:c1:00.4 after resume to restore Goodix fingerprint reader.

PCI_FUNC="0000:c1:00.4"
GOODIX_ID="27c6:609c"
DRIVER_PATH="/sys/bus/pci/drivers/xhci_hcd"
logger -t fp-rebind "Running after wake script for Goodix fingerprint reader"
logger -t fp-rebind "Checking PCI function $PCI_FUNC for Goodix device ID $GOODIX_ID"
# Give the system a moment to resume normally
sleep 2
# Check if the fingerprint reader is missing
if ! lsusb -d "$GOODIX_ID" >/dev/null 2>&1; then
  logger -t fp-rebind "Goodix missing after resume, resetting xHCI controller $PCI_FUNC"
  # Unbind and rebind only that PCI function
  echo "$PCI_FUNC" >"$DRIVER_PATH/unbind"
  sleep 1
  echo "$PCI_FUNC" >"$DRIVER_PATH/bind"
  sleep 2
  # Restart fprintd so it picks up the reader again
  systemctl try-restart fprintd.service
else
  logger -t fp-rebind "Fingerprint sensor appears available on the PCI bus, nothing to do."
fi
```

Don't forget it needs to be executable:

```sh
# Make the script executable
> sudo chmod +x /etc/local/bin/run-after-wake-for-fprint.sh
```

### Create a systemd unit to run the script when needed

```sh
> sudo vim /etc/systemd/system/run-after-wake-for-fprint.service
```

Give it the following content:

```conf
[Unit]
Description=Run custom script after resume to restart fingerprint sensor
After=sleep.target
Wants=sleep.target

[Service]
Type=oneshot
ExecStart=/etc/local/bin/run-after-wake-for-fprint.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=sleep.target
```

### Start the service

Once the service is enabled, it will execute on each wake event:

```sh
# Start the service
> sudo systemctl enable --now run-after-wake-for-fprint.service
```
