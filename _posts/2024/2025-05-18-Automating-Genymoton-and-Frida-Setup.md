---
title: "Automating Genymoton and Frida Setup"
date: 2025-05-18 00:25:55 +0300
categories: [Scripting, bash]
tags: [virtualization, scripting, bash]
image: /assets/img/vm/vbox-logo.png
alt: "Automating Genymoton and Frida Setup"
---

### Automating the BurpSuite + Genymotion + frida Setup

Like most things, it begins quietly.
A couple of evenings that were stolen. An interest in DevOps. Automation, however, is not silent. It expands. It draws you in. A sleeping instinct that waits for authorization to awaken is more than just a skill.
And I comply.

Developers have been all around me lately, pushing code, shipping quickly, and breaking things more quickly. I watch from the sidelines, curious. And in that confusion, I ended up in a position I have found myself so oftenly: **testing mobile applications**.

Long setup times and broken environments caused subtle but persistent pain in developers, and quite frankly, me. Repetition wastes time. And dont we all detest wasting time.

I then thought a solution in a whisper: partial automation.

Not all of it. Not just yet. Just enough to facilitate the process, quicker, managed.
Because the best solutions don't always stand out.
They hum softly in the background, resembling obsession disguised, because sometimes, the best fixes don’t scream.
They hum quietly in the background — like obsession dressed as efficiency

## Tools of trade
Pretty standard.
- Burp
- Genymotion
- Frida for dynamic intrumentation.
- MacOS (can be enhanced for linux too)

Setting things up... takes time. Too much time. Click here, type that, wait. Again. And again.
But what if it didn’t?

Lately, I can’t unsee it — automation. It’s everywhere. In the repetition. In the patterns. In the silence between commands.
Endless possibilities, all whispering the same thing: "Do less. Achieve more."

It only took two macOS machines — same tools, same tedious routine — to show me the truth. That this wasn’t just setup. It was a challenge. An invitation.
To automate. To reclaim time.

## Automation with bash.
Clearly bash had to be a first go to. This will later on be dockerised, but for now, just see if this is easy enough to do.

### 1. Preliquisites.
Some assamtions have to be made. Make sure the following are already downloaded.
- Genymotion
- BurpSuite Pro/Community
- Make sure BurpSuite is running and listener set.

### 2. Into it

A little color never hurt anyone
```bash
# Color codes for better output visibility
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color
```

### 3. Cleanup Function

It uses `gmtool`, which is silent and effective, to inventory and list virtual machines. It then checks to see if a virtual machine is running, something that most people lack the courage to do.
If it is, it's over.

Not in a dramatic way. Not by force.
Just a quiet, purposeful shutdown.
```bash
cleanup() {
    print_info "Performing cleanup..."
    
    # Stop the VM if it's running and we have its UUID
    if [ -n "$CUSTOM_PHONE_UUID" ] && [ -f "$GMTOOL_PATH" ]; then
        VM_STATUS=$("$GMTOOL_PATH" admin list 2>/dev/null | grep -F "$CUSTOM_PHONE_UUID" | awk -F'|' '{gsub(/^[[:space:]]+|[[:space:]]+$/, "", $1); print $1}')
        
        if [ "$VM_STATUS" = "Running" ] || [ "$VM_STATUS" = "On" ]; then
            print_info "Stopping Custom Phone VM..."
            "$GMTOOL_PATH" admin stop "$CUSTOM_PHONE_UUID"
            if [ $? -eq 0 ]; then
                print_success "Custom Phone VM stopped successfully."
            else
                print_warning "Failed to stop Custom Phone VM. You may need to stop it manually."
            fi
        else
            print_info "Custom Phone VM is not running. No need to stop it."
        fi
    fi
    
    # Clean up any temporary files
    if [ -d "$temp_dir" ]; then
        rm -rf "$temp_dir"
        print_info "Temporary files cleaned up."
    fi
    
    print_info "Cleanup completed. Exiting..."
}

```

### 4. Check if burpsuite is running on default port
Checks if Burp Suite is running.
This isn’t just a check — it’s surveillance. Quiet confirmation that the interceptor is alive.

After all, what’s the point of launching anything if Burp isn’t watching?
```bash
check_burpsuite() {
    print_info "Checking if Burp Suite is running..."
    
    # Check if port 8080 is in use (default Burp Suite proxy port)
    if lsof | grep -i "Burpsuite" > /dev/null 2>&1 || lsof | grep -i "Burp suite" > /dev/null 2>&1; then
        print_success "Burp Suite appears to be running on port 8080."
        return 0
    else
        print_error "Burp Suite does not appear to be running on port 8080. Please start Burp Suite before continuing."
        return 1
    fi
}
```

### 5. check if portswigger CA is installed on device
```bash
install_ca_cert() {
    local cert_path="$1"
    
    if [ ! -f "$cert_path" ]; then
        print_error "Certificate file not found: $cert_path"
        return 1
    fi
    
    print_info "Installing PortSwigger CA certificate on the device..."
    
    # Create a readable filename
    local cert_filename=$(basename "$cert_path")
    local device_cert_path="/sdcard/$cert_filename"
    
    # Push certificate to device
    print_info "Copying certificate to device..."
    adb push "$cert_path" "$device_cert_path"
    if [ $? -ne 0 ]; then
        print_error "Failed to copy certificate to device."
        return 1
    fi
    
    # For Android 7+, user needs to manually install the certificate
    print_info "Please follow these instructions on your device:"
    print_info "1. The Settings app should open automatically to the certificate installation page"
    print_info "2. If prompted, name the certificate 'Burp Suite PortSwigger'"
    print_info "3. Select 'VPN and apps' or 'CA Certificate' when prompted for certificate type"
    print_info "4. Confirm any security warnings"
    
    # Launch certificate installer on the device
    adb shell "am start -n com.android.settings/.security.InstallCaCertificateWarningActivity"
    sleep 2
    adb shell "am start -a android.settings.SECURITY_SETTINGS"
    sleep 2
    
    # Prompt user to confirm installation is complete
    read -p "Press Enter after you have completed the certificate installation on the device... " -r
    
    # Verify certificate installation
    print_info "Verifying certificate installation..."
    if check_portswigger_cert; then
        print_success "PortSwigger CA certificate installed successfully."
        return 0
    else
        print_warning "Could not verify PortSwigger certificate installation. Please check device manually."
        read -p "Did you complete the certificate installation on the device? (y/n): " cert_installed
        if [[ "$cert_installed" =~ ^[Yy]$ ]]; then
            return 0
        else
            return 1
        fi
    fi
}
```
### 5. Check if portswigger cert is installed
```sh
check_portswigger_cert() {
    # Check for PortSwigger certificate in system CA store
    local cert_check=$(adb shell "grep -i portswigger /system/etc/security/cacerts/* 2>/dev/null || echo ''")
    
    # Check for PortSwigger certificate in user CA store
    local user_cert_check=$(adb shell "grep -i portswigger /data/misc/user/0/cacerts-added/* 2>/dev/null || echo ''")
    
    # Try to check certificate list using security command
    local security_check=$(adb shell "security list 2>/dev/null | grep -i portswigger || echo ''")
    
    if [ -n "$cert_check" ] || [ -n "$user_cert_check" ] || [ -n "$security_check" ]; then
        return 0  # PortSwigger certificate found
    else
        return 1  # PortSwigger certificate not found
    fi
}
```

### 6. Check if genymotion is installed
I look, i probe. Just to check if genymotion is installed on macos. And sure it might not be installed, but you dont care now, do you script? You sit there knowing i will likely install it manually. And you are right.
```sh
check_genymotion() {
    print_info "Checking if Genymotion is installed..."
    
    # Check for Genymotion in the common macOS application location
    if [ -d "/Applications/Genymotion.app" ]; then
        print_success "Genymotion is installed."
        GMTOOL_PATH="/Applications/Genymotion.app/Contents/MacOS/gmtool"
        
        if [ ! -f "$GMTOOL_PATH" ] || [ ! -x "$GMTOOL_PATH" ]; then
            print_error "gmtool not found at $GMTOOL_PATH or not executable."
        fi
        
        return 0
    else
        print_error "Genymotion is not installed in /Applications. Please install Genymotion first."
        return 1
    fi
}
```
### 7. List vms
You run `gmtool`. Not to show off. Just to see what’s there. The VMs — downloaded, waiting, like actors backstage. I see what you want me to see. Bravo, you!
```sh
list_vms() {
    print_info "Listing available Genymotion VMs..."
    
    VM_LIST=$("$GMTOOL_PATH" admin list)
    if [ $? -ne 0 ]; then
        print_error "Failed to list VMs. Please check if Genymotion is running properly."
    fi
    
    echo "$VM_LIST"
}
```

### 8. Check custom VM
You now let me know the VMs running. I see what you are doing. Willing to share more about them. But i'll take just that, for now.
```sh
get_custom_phone_vm() {
    print_info "Checking for 'Custom Phone' VM..."
    
    CUSTOM_PHONE_INFO=$("$GMTOOL_PATH" admin list | grep "Custom Phone")
    if [ -z "$CUSTOM_PHONE_INFO" ]; then
        print_error "Custom Phone VM not found. Please install it before continuing."
    fi
    
    # Extract the UUID - it's the third column in the pipe-separated table
    # The format is: State | ADB Serial | UUID | Name
    CUSTOM_PHONE_UUID=$(echo "$CUSTOM_PHONE_INFO" | awk -F'|' '{gsub(/^[[:space:]]+|[[:space:]]+$/, "", $3); print $3}')
    if [ -z "$CUSTOM_PHONE_UUID" ]; then
        print_error "Failed to extract Custom Phone VM UUID."
    fi
    
    print_success "Found Custom Phone VM with UUID: $CUSTOM_PHONE_UUID"
    return 0
}
```

### 9. ADB connect
Ohh ADB, you like the wait. Because the device is booting, breathing, becoming. And when it’s ready, you’re already there.
Connected. Watching.
```sh
wait_for_adb_connection() {
    print_info "Waiting for ADB connection to device (this may take a minute)..."
    
    # Restart ADB server to ensure clean connection state
    print_info "Restarting ADB server..."
    adb kill-server
    sleep 2
    adb start-server
    sleep 3
    
    # Get ADB port from VM info
    ADB_PORT=$(echo "$CUSTOM_PHONE_INFO" | awk -F'|' '{gsub(/^[[:space:]]+|[[:space:]]+$/, "", $2); print $2}' | cut -d':' -f2)
    
    if [ -z "$ADB_PORT" ]; then
        print_warning "Could not determine ADB port from VM info. Using default port 6555."
        ADB_PORT="6555"
    fi
    
    # Explicitly connect to the device
    print_info "Connecting to device at 127.0.0.1:$ADB_PORT..."
    adb connect 127.0.0.1:$ADB_PORT
    
    # Wait a moment for the connection to establish
    sleep 3
    
    # Check for ADB availability with timeout
    MAX_ATTEMPTS=30
    ATTEMPT=0
    
    while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
        # Check both generic device connection and specific emulator connection
        DEVICE_CHECK=$(adb devices | grep -v "List" | grep -v "offline" | grep "device$")
        
        if [ -n "$DEVICE_CHECK" ]; then
            print_success "ADB is now connected to the device."
            print_info "Connected device(s):"
            adb devices | grep -v "List"
            
            # Additional wait to ensure system is fully booted
            sleep 5
            return 0
        fi
        
        # If no connection yet, try connecting again after a few attempts
        if [ $((ATTEMPT % 5)) -eq 0 ] && [ $ATTEMPT -gt 0 ]; then
            print_info "Retrying connection to 127.0.0.1:$ADB_PORT..."
            adb connect 127.0.0.1:$ADB_PORT
        fi
        
        ATTEMPT=$((ATTEMPT+1))
        print_info "Waiting for ADB connection... ($ATTEMPT/$MAX_ATTEMPTS)"
        sleep 5
    done
    
    print_error "Timed out waiting for ADB connection. Please check the VM and ADB setup manually."
}
```
### 10. Start the Custom VM
Launch the Custom Phone VM, whenever you are ready.
```sh
start_custom_phone_vm() {
    print_info "Checking VM status..."
    
    # Check if VM is already running - Genymotion might report "Running" or "On"
    VM_STATUS=$("$GMTOOL_PATH" admin list | grep -F "$CUSTOM_PHONE_UUID" | awk -F'|' '{gsub(/^[[:space:]]+|[[:space:]]+$/, "", $1); print $1}')
    
    # VM is already running
    if [ "$VM_STATUS" = "Running" ] || [ "$VM_STATUS" = "On" ]; then
        print_success "Custom Phone VM is already running."
        # Still need to ensure ADB is connected even if VM is already running
        wait_for_adb_connection
        return 0
    fi
    
    # VM needs to be started
    print_info "Starting Custom Phone VM..."
    START_OUTPUT=$("$GMTOOL_PATH" admin start "$CUSTOM_PHONE_UUID" 2>&1)
    START_RESULT=$?
    
    # Check for both exit code and "already started" message
    if [ $START_RESULT -ne 0 ] && ! echo "$START_OUTPUT" | grep -q "already started"; then
        print_error "Failed to start Custom Phone VM. Please check Genymotion setup."
    elif echo "$START_OUTPUT" | grep -q "already started"; then
        print_success "Custom Phone VM is already running."
        # Still need to ensure ADB is connected even if VM is already started
        wait_for_adb_connection
        return 0
    fi
    
    # VM was just started, wait for it to boot and connect to ADB
    print_info "VM started successfully. Waiting for it to boot..."
    # Initial delay to let VM start booting
    sleep 10
    
    # Wait for ADB connection
    wait_for_adb_connection
}
```

### 11. Check ADB device connection
You check if ADB is connected and devices can be reached. You care, for the user who runs you. I like that.
```sh
check_device_connection() {
    print_info "Checking for connected devices..."
    devices=$(adb devices | grep -v "List" | grep "device$")
    if [ -z "$devices" ]; then
        print_error "No devices connected. Please connect an Android device and try again."
    else
        print_success "Device found!"
        echo "$devices"
    fi
}
```
### 12. Frida Server Setup
Another configuration is the Frida server.  You don't guess, to start. It is aware. It selects the ideal Frida server binary by examining the macOS machine's CPU architecture; no steps are wasted, and no errors are made.

Next the frida server is copied to the enulated device and started.
```sh
# Set permissions
print_info "Setting permissions on Frida server..."
chmod 755 "$temp_dir/$frida_server_file"

# Push to device
print_info "Uploading Frida server to device..."
adb push "$temp_dir/$frida_server_file" /data/local/tmp/frida-server
adb shell "chmod 755 /data/local/tmp/frida-server"

# Check if upload was successful
if adb shell "[ -f /data/local/tmp/frida-server ] && echo 'yes'" | grep -q "yes"; then
    print_success "Frida server uploaded successfully."
else
    print_error "Failed to upload Frida server to the device."
fi

# Start Frida server in the background
print_info "Starting Frida server on the device..."
adb shell "su -c 'killall frida-server 2>/dev/null || true'"
adb shell "su -c '/data/local/tmp/frida-server &'" &
```
### 13. Burpsuite and Android Device Proxy Configuration
The script will prompt the user to configure burpsuite's proxy settings. The script will get the proxy settings and ask the user to set the details on the android vm's network proxy settings.
```sh
# Ask for proxy port
read -p "Enter the proxy port number (default: 8080): " proxy_port
proxy_port=${proxy_port:-8080}

print_info "Please configure your Android device to use the following proxy settings:"
print_info "Host: $host_ip"
print_info "Port: $proxy_port"
print_info "Go to Settings -> Network & Internet -> Wi-Fi -> [Your connected network] -> Edit -> Advanced options -> Proxy Manual"

read -p "Press Enter when you have configured the proxy settings... " -r
```
### 14. Package analysis
The script will list the package to use and once entered, frida will now start it using the `multiple-unpinning` js file to bypass the android TrustManager's security implementation.
```sh
# List available packages
print_info "Listing available packages on the device..."
packages=$(adb shell pm list packages | cut -d: -f2)

# Filter packages containing "to-be-replaced-package-name"
theapp_packages=$(echo "$packages" | grep -i "to-be-replaced-package-name")

if [ -z "$theapp_packages" ]; then
    print_info "No packages containing 'to-be-replaced-package-name' found. Listing all packages instead."
    # List all packages and ask user to select
    package_array=()
    i=1
    while IFS= read -r line; do
        package_array+=("$line")
        echo "$i) $line"
        ((i++))
    done <<< "$packages"
else
    print_info "Found packages containing 'to-be-replaced-package-name':"
    package_array=()
    i=1
    while IFS= read -r line; do
        package_array+=("$line")
        echo "$i) $line"
        ((i++))
    done <<< "$theapp_packages"
fi

# Ask user to select a package
read -p "Enter the number of the package you want to target: " package_number

if ! [[ "$package_number" =~ ^[0-9]+$ ]] || [ "$package_number" -lt 1 ] || [ "$package_number" -gt ${#package_array[@]} ]; then
    print_error "Invalid selection. Please enter a number between 1 and ${#package_array[@]}."
fi

selected_package=${package_array[$((package_number-1))]}
print_info "Selected package: $selected_package"

# Launch the selected app with Frida
print_info "Launching the app with Frida multiple unpinning script..."

# Check if frida CLI command is available
if ! command_exists frida; then
    print_warning "Frida CLI command not found, even though Frida packages are installed."
    print_info "This could be due to PATH issues. Please provide the path to the frida command."
    read -p "Enter the path to the frida command (or press Enter to continue with default): " frida_path
    
    if [ -n "$frida_path" ]; then
        if [ -f "$frida_path" ] && [ -x "$frida_path" ]; then
            FRIDA_CMD="$frida_path"
            print_success "Using provided Frida command: $FRIDA_CMD"
        else
            print_error "The provided path does not exist or is not executable: $frida_path"
        fi
    else
        print_info "Continuing with default 'frida' command. If this fails, run the script again and provide the correct path."
        FRIDA_CMD="frida"
    fi
else
    print_success "Frida CLI command is available."
    FRIDA_CMD="frida"
fi

# Launch the app with Frida
print_info "Running Frida with the selected package..."
frida_cmd="$FRIDA_CMD -U -f '$selected_package' --codeshare akabe1/frida-multiple-unpinning"
print_info "Executing: $frida_cmd"

eval $frida_cmd

# Check if Frida command executed successfully
if [ $? -eq 0 ]; then
    print_success "Frida launched successfully! The app should be running with SSL unpinning."
    print_success "Setup completed successfully!"
else
    print_error "Failed to launch Frida. Check if the device is properly connected and Frida server is running."
fi
```

The full script will be in the github repo herenym

This can be confirmed from the VirtualBox manager interface.
![error](/assets/img/vm/vbox.png)

First step in my devops learning path :-)