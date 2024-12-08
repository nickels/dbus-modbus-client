# dbus-modbus-client

Reads data from Modbus devices and publishes on D-Bus.  Service names and
paths are per the [Victron D-Bus specification](https://github.com/victronenergy/venus/wiki/dbus).

Modbus devices using the RTU, TCP, and UDP transports are supported.

## VregLink

With some devices, the [VregLink](https://github.com/victronenergy/venus/wiki/dbus-api#the-vreglink-interface)
interface is supported.

The VregLink interface uses a block of up to 125 Modbus holding registers
accessed with the Modbus Read/Write Multiple Registers function (code 23).
Byte data is packed into 16-bit register values MSB first. All accesses
start from offset zero in the VregLink block. The base address is device
specific.

### Write

| Offset | Field   |
| ------ | ------- |
| 0      | VREG ID |
| 1      | Size    |
| 2..N   | Data    |

A write operation contains the VREG to access and, optionally, data for
setting the value. The size field indicates the length in bytes of the
data and must be 2x the number of registers or 1 less. To select a VREG
for a subsequent read, only the first field (VREG ID) should be present.

### Read

| Offset | Field   |
| ------ | ------- |
| 0      | VREG ID |
| 1      | Status  |
| 2      | Size    |
| 3..N   | Data    |

A read operation returns the value of the selected VREG or an error code if
the access failed. The size field indicates the length of the data in bytes.
The actual size of the VREG value is returned even if the requested number
of registers is too small to contain it.

### Status

The following status codes are possible.

| Value  | Meaning                      |
| ------ | ---------------------------- |
| 0      | Success                      |
| 0x8000 | Read: unknown error          |
| 0x8001 | Read: VREG does not exist    |
| 0x8002 | Read: VREG is write-only     |
| 0x8100 | Write: unknown error         |
| 0x8101 | Write: VREG does not exist   |
| 0x8102 | Write: VREG is read-only     |
| 0x8104 | Write: data invalid for VREG |

### Examples

If the VregLink register block begins at address 0x4000, then to read
VREG 0x100 (product ID), the following Modbus transaction would be used.

| Field Name                | Hex | Comment                       |
| ------------------------- | --- | ----------------------------- |
| _Request_                 |     |                               |
| Function                  | 17  | Read/Write Multiple Registers |
| Read Starting Address Hi  | 40  | VregLink base address         |
| Read Starting Address Lo  | 00  |                               |
| Quantity to Read Hi       | 00  |                               |
| Quantity to Read Lo       | 05  |                               |
| Write Starting Address Hi | 40  | VregLink base address         |
| Write Starting address Lo | 00  |                               |
| Quantity to Write Hi      | 00  |                               |
| Quantity to Write Lo      | 01  |                               |
| Write Byte Count          | 02  |                               |
| Write Register Hi         | 01  | PRODUCT_ID                    |
| Write Register Lo         | 00  |                               |
| _Response_                |     |                               |
| Function                  | 17  | Read/Write Multiple Registers |
| Byte Count                | 0A  |                               |
| Read Register Hi          | 01  | PRODUCT_ID                    |
| Read Register Lo          | 00  |                               |
| Read Register Hi          | 00  | Status: success               |
| Read Register Lo          | 00  |                               |
| Read Register Hi          | 00  | Size: 4                       |
| Read Register Lo          | 04  |                               |
| Read Register Hi          | 00  | Product ID                    |
| Read Register Lo          | 12  |                               |
| Read Register Hi          | 34  |                               |
| Read Register Lo          | FE  |                               |

To set VREG 0x10C (description), the Modbus transaction might look as
follows.

| Field Name                | Hex | Comment                       |
| ------------------------- | --- | ----------------------------- |
| _Request_                 |     |                               |
| Function                  | 17  | Read/Write Multiple Registers |
| Read Starting Address Hi  | 40  | VregLink base address         |
| Read Starting Address Lo  | 00  |                               |
| Quantity to Read Hi       | 00  |                               |
| Quantity to Read Lo       | 02  |                               |
| Write Starting Address Hi | 40  | VregLink base address         |
| Write Starting address Lo | 00  |                               |
| Quantity to Write Hi      | 00  |                               |
| Quantity to Write Lo      | 08  |                               |
| Write Byte Count          | 10  |                               |
| Write Register Hi         | 01  | DESCRIPTION1                  |
| Write Register Lo         | 0C  |                               |
| Write Register Hi         | 00  | Size: 11                      |
| Write Register Lo         | 0B  |                               |
| Write Register Hi         | 4D  | 'M'                           |
| Write Register Lo         | 79  | 'y'                           |
| Write Register Hi         | 20  | ' '                           |
| Write Register Lo         | 50  | 'P'                           |
| Write Register Hi         | 72  | 'r'                           |
| Write Register Lo         | 65  | 'e'                           |
| Write Register Hi         | 63  | 'c'                           |
| Write Register Lo         | 69  | 'i'                           |
| Write Register Hi         | 6f  | 'o'                           |
| Write Register Lo         | 75  | 'u'                           |
| Write Register Hi         | 73  | 's'                           |
| Write Register Lo         | 00  | Padding                       |
| _Response_                |     |                               |
| Function                  | 17  | Read/Write Multiple Registers |
| Byte Count                | 04  |                               |
| Read Register Hi          | 01  | DESCRIPTION1                  |
| Read Register Lo          | 0C  |                               |
| Read Register Hi          | 00  | Status: success               |
| Read Register Lo          | 00  |                               |

## Steps to Implement Custom Modbus Client Registration

### 1. Create a Script to Register the Custom Client

Save the following script as `/data/scripts/register_custom_modbus_client.sh`. This script will:

- Clone or update a Git repository containing your custom client.
- Symlink the custom client to the appropriate location (`/opt/victronenergy/dbus-modbus-client/`).

```bash
#!/bin/bash

# Set working directory
WORK_DIR="/data/dbus-modbus-client"
CUSTOM_SCRIPT_URL="<<<YOUR GIT REPO HERE>>>"
CUSTOM_CLIENT_NAME="carlo_gavazzi.py"

# Ensure git is installed (you might need to install git if not present)
if ! command -v git &> /dev/null; then
    echo "git not found, installing..."
    opkg update && opkg install git
fi

# Check if the repository already exists
if [ ! -d "$WORK_DIR" ]; then
    echo "Cloning the repository..."
    git clone "$CUSTOM_SCRIPT_URL" "$WORK_DIR"
else
    echo "Repository already exists. Pulling latest changes..."
    cd "$WORK_DIR" && git pull
fi

# Check if the custom client exists in the repo, if not, exit with error
if [ ! -f "$WORK_DIR/$CUSTOM_CLIENT_NAME" ]; then
    echo "Error: Custom client script '$CUSTOM_CLIENT_NAME' not found in repository."
    exit 1
fi

# Register custom client
echo "Registering the custom client..."
ln -sf "$WORK_DIR/$CUSTOM_CLIENT_NAME" "/opt/victronenergy/dbus-modbus-client/"

# Re-register dbus-modbus-client.py
echo "Registering dbus-modbus-client.py..."
ln -sf "$WORK_DIR/dbus-modbus-client.py" "/opt/victronenergy/dbus-modbus-client/dbus-modbus-client.py"

echo "Custom client registration complete."
```

### 2. Make the Script Executable

After creating the script, ensure it is executable by running:

```bash
chmod +x /data/scripts/register_custom_modbus_client.sh
```

### 3. Create a Boot Hook (`/data/rc.local`)

VenusOS does not persist `/etc/rc.local` across updates, but `/data/rc.local` is persistent. To ensure the custom client is registered on every boot, create the following file:

```bash
nano /data/rc.local
```

Add this content to the file:

```bash
#!/bin/sh
/data/scripts/register_custom_modbus_client.sh
exit 0
```

### 4. Make `/data/rc.local` Executable

Finally, ensure that `/data/rc.local` is executable by running:

```bash
chmod +x /data/rc.local
```

### Important Note: Syncing with the Original Repo

If you are using a forked version of the `dbus-modbus-client`, it’s important to keep it in sync with the original repository. This will ensure that you benefit from any updates, bug fixes, and improvements made to the core `dbus-modbus-client` code. You can periodically pull changes from the original repo into your fork to stay up to date.

