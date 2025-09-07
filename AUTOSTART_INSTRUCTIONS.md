# Conky Autostart Setup Guide

This guide explains how to set up Conky to start automatically with all its scripts when you log in to your desktop environment.

## Prerequisites
- Conky is already installed on your system
- Configuration files are located in `/root/CascadeProjects/conky/`
- Required dependencies: `conky-all` (or `conky`), `lm-sensors`

## Required Scripts

### 1. Power Monitor Script
This setup includes a power monitoring script that needs to be installed:

```bash
# First, install required dependencies
sudo apt-get update
sudo apt-get install -y bc lm-sensors

# Create the script directory if it doesn't exist
sudo mkdir -p /usr/local/bin/
sudo tee /usr/local/bin/power_monitor_rapl.sh > /dev/null << 'EOL'
#!/bin/bash
# Power monitor script for Intel RAPL

while true; do
    # Check if the RAPL power files exist
    if [ -d "/sys/class/powercap/intel-rapl/" ]; then
        # Sum up power usage from all RAPL domains
        power_mw=0
        for domain in /sys/class/powercap/intel-rapl:*/energy_uj; do
            if [ -f "$domain" ]; then
                # Read the energy in microjoules and convert to watts
                energy1=$(cat "$domain")
                sleep 1
                energy2=$(cat "$domain")
                power_mw=$((power_mw + (energy2 - energy1) / 1000))
            fi
        done
        # Convert milliwatts to watts with 2 decimal places
        power_w=$(echo "scale=2; $power_mw/1000" | bc)
        echo $power_w > /tmp/power_usage
    else
        echo "0" > /tmp/power_usage
    fi
    sleep 1
done
EOL

# Make the script executable
sudo chmod +x /usr/local/bin/power_monitor_rapl.sh

# Create a systemd service to run the power monitor
sudo tee /etc/systemd/system/power-monitor.service > /dev/null << 'EOL'
[Unit]
Description=Power Monitor Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/power_monitor_rapl.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOL

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable --now power-monitor.service
```

### 2. Lua and Configuration Files
Copy the necessary files to the user's configuration directory:

```bash
# For the current user (non-root):
CONFIG_DIR="$HOME/.config/conky"
mkdir -p "$CONFIG_DIR"

# Copy the configuration files
cp /root/CascadeProjects/conky/conky.conf "$CONFIG_DIR/"
cp /root/CascadeProjects/conky/rings.lua "$CONFIG_DIR/"

# Set proper permissions
chmod 755 "$CONFIG_DIR"
chmod 644 "$CONFIG_DIR/conky.conf"
chmod 644 "$CONFIG_DIR/rings.lua"

# For root user (if needed):
# sudo mkdir -p /root/.config/conky/
# sudo cp /root/CascadeProjects/conky/conky.conf /root/.config/conky/
# sudo cp /root/CascadeProjects/conky/rings.lua /root/.config/conky/
```

## Autostart Methods

### Method 1: Using .desktop File (Recommended)

1. **Create or update the desktop file** for the current user:
   ```bash
   # Create autostart directory if it doesn't exist
   AUTOSTART_DIR="$HOME/.config/autostart"
   mkdir -p "$AUTOSTART_DIR"
   
   # Create the desktop file
   cat > "$AUTOSTART_DIR/conky.desktop" << EOL
[Desktop Entry]
Type=Application
Name=Conky
Comment=System monitoring tool
Exec=conky -c $HOME/.config/conky/conky.conf
StartupNotify=false
Terminal=false
X-GNOME-Autostart-enabled=true
EOL

   # Set proper permissions
   chmod 644 "$AUTOSTART_DIR/conky.desktop"
   ```

   For root user (not recommended for security reasons):
   ```bash
   # sudo mkdir -p /root/.config/autostart/
   # sudo bash -c 'cat > /root/.config/autostart/conky.desktop << EOL
[Desktop Entry]
Type=Application
Name=Conky
Comment=System monitoring tool
Exec=sudo -u $SUDO_USER env DISPLAY=$DISPLAY DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS conky -c /root/.config/conky/conky.conf
StartupNotify=false
Terminal=false
EOL'
   ```

2. **Make the file executable**:
   ```bash
   chmod +x ~/.config/autostart/conky.desktop
   ```

### Method 2: Using ~/.xprofile (Alternative)

1. Edit or create the `~/.xprofile` file:
   ```bash
   echo 'conky -c /root/CascadeProjects/conky/conky.conf &' >> ~/.xprofile
   ```

### Method 3: System-wide Autostart (for all users)

1. **Create a system desktop file**:
   ```bash
   sudo bash -c 'cat > /etc/xdg/autostart/conky.desktop <<EOL
   [Desktop Entry]
   Type=Application
   Name=Conky
   Comment=System monitoring tool
   Exec=conky -c /root/CascadeProjects/conky/conky.conf
   StartupNotify=false
   Terminal=false
   EOL'
   ```

## Verifying the Setup

1. First, ensure the power monitor service is running:
   ```bash
   sudo systemctl status power-monitor.service
   ```

2. Check the power reading:
   ```bash
   cat /tmp/power_usage
   ```

3. Start Conky manually to test:
   ```bash
   killall conky 2>/dev/null
   conky -c /root/CascadeProjects/conky/conky.conf &
   ```

4. Check if Conky is running:
   ```bash
   pgrep -l conky
   ```

## Troubleshooting

- **Conky doesn't start**
  Check the log:
  ```bash
  tail -f ~/.cache/conky/conky.log
  ```

- **Power monitoring not working**
  Check if the power monitor service is running:
  ```bash
  sudo systemctl status power-monitor.service
  sudo journalctl -u power-monitor.service -f
  ```
  
  Check if the temporary file is being updated:
  ```bash
  watch -n1 cat /tmp/power_usage
  ```

- **Permission errors**
  Ensure the config files are readable:
  ```bash
  chmod -R 755 /root/CascadeProjects/conky/
  ```
  
  Make sure the power monitor script is executable:
  ```bash
  sudo chmod +x /usr/local/bin/power_monitor_rapl.sh
  ```

- **Rings not displaying**
  Ensure the Lua script is in the correct location and has proper permissions:
  ```bash
  ls -l ~/.config/conky/rings.lua
  chmod 644 ~/.config/conky/rings.lua
  ```

## File Locations

### User-Specific Files (per user)
- **Configuration Directory**: `$HOME/.config/conky/`
  - Main config: `$HOME/.config/conky/conky.conf`
  - Lua script: `$HOME/.config/conky/rings.lua`
- **Autostart Entry**: `$HOME/.config/autostart/conky.desktop`
- **Logs**: `$HOME/.cache/conky/conky.log`

### System-Wide Files
- **Power Monitor Script**: `/usr/local/bin/power_monitor_rapl.sh`
- **Power Monitor Service**: `/etc/systemd/system/power-monitor.service`
- **Temporary Power Data**: `/tmp/power_usage`

### Source Files (Backup/Reference)
- All original files are stored in: `/root/CascadeProjects/conky/`

- **Important Paths**:
  - Power reading: `/tmp/power_usage`
  - Logs: `~/.cache/conky/conky.log`

- **Customization**:
  - Edit `rings.lua` to modify the clock and power rings appearance
  - Adjust update intervals in `conky.conf` if needed
  - The power monitor script can be modified to support different hardware by adjusting the RAPL paths

## Customization
Edit the `conky.conf` file to modify the appearance and behavior of Conky. After making changes, you can either restart your session or run:

```bash
killall conky && conky -c /root/CascadeProjects/conky/conky.conf &
```
