
# Table of Contents

1. Introduction
2. System Overview  
3. Pre-requisites
4. Initial Synchronization: Server 106 to Server 254
   - 4.1. Directory Structure
   - 4.2. Configuration File
   - 4.3. Python Synchronization Script
   - 4.4. Systemd Service Setup
   - 4.5. Initial Synchronization Steps
5. Failover Synchronization: Server 254 to Server 106
   - 5.1. Directory Structure
   - 5.2. Configuration File
   - 5.3. Python Synchronization Script
   - 5.4. Systemd Service Setup
   - 5.5. Failover Steps
6. Testing the Synchronization
7. Troubleshooting
8. Best Practices and Recommendations
9. Conclusion



----------

## **1. Introduction**

This guide provides a detailed walkthrough for setting up and managing a **bidirectional file synchronization** system between two servers:

-   **Server 106**: Initially designated as the **Primary Source**.
-   **Server 254**: Initially designated as the **Secondary Destination**.

The synchronization ensures that files are consistently mirrored between the two servers, with an automated **failover mechanism** that promotes the secondary server to primary in case of failure, maintaining data integrity and availability.

----------

## **2. System Overview**

-   **Primary Synchronization (106 ➔ 254)**:  
    Synchronizes files from **Server 106** to **Server 254**.
    
-   **Failover Synchronization (254 ➔ 106)**:  
    In the event of **Server 106** failure, synchronizes files from **Server 254** back to **Server 106**.
    
-   **State Management**:  
    Utilizes a `state.json` file to track synchronized files, preventing redundant transfers.
    
-   **Automation**:  
    Employs a Python script with `watchdog` for real-time monitoring and `rsync` for efficient file synchronization, managed by `systemd` services.
    

----------

## **3. Pre-requisites**

Before proceeding, ensure the following:

-   **Operating System**: Both servers run a Unix-like OS (e.g., Linux).
    
-   **Python 3**: Installed on both servers.
    
-   **rsync**: Installed on both servers.
    
-   **SSH Access**: Password-less SSH key-based authentication set up from both servers.
    
-   **Root or Sudo Access**: Required for configuration and service setup.
    
-   **Dependencies**:  
    Install necessary Python packages on both servers:
    
    ```bash
    sudo apt-get update
    sudo apt-get install python3-pip -y
    pip3 install watchdog
    
    ```
    
-   **Directory Structure**:  
    Ensure that the necessary directories exist or create them as per the following sections.
    

----------

## **4. Initial Synchronization: Server 106 to Server 254**

### **4.1. Directory Structure**

**Server 106 (Primary Source):**

```bash
/opt/opennms/
├── rrd_data_move/
│   └── [Your Files]
└── rrd_monitoring/
    ├── config_rrdPyScript.json
    ├── rrdPyScript.py
    ├── rrdPyScript.json
    └── rrdPyScript.log

```

**Server 254 (Secondary Destination):**

```bash
/opt/opennms/
├── rrd_data_move/
│   └── [Mirrored Files]
└── rrd_monitoring/
    ├── config_rrdPyScript.json
    ├── rrdPyScript.py
    ├── rrdPyScript.json
    └── rrdPyScript.log

```

### **4.2. Configuration File**

**Path:** `/opt/opennms/rrd_monitoring/config_rrdPyScript.json`

**Content:**

```json
{
    "source_dir": "/opt/opennms/rrd_data_move/",
    "state_file": "/opt/opennms/rrd_monitoring/rrdPyScript.json",
    "remote_sync": {
        "remote_user": "root",
        "remote_host": "192.168.29.254",
        "remote_dest_dir": "/opt/opennms/rrd_data_move/",
        "ssh_key_path": "/root/.ssh/id_rsa_rrd_sync_primary"
    },
    "logging": {
        "log_file": "/opt/opennms/rrd_monitoring/rrdPyScript.log",
        "log_level": "INFO"
    },
    "debounce_time": 5,
    "monitored_extensions": [
        ".rrd",
        ".txt",
        ".properties",
        ".meta"
    ],
    "rsync_options": {
        "archive": true,
        "compress": true,
        "verbose": true,
        "update": true,
        "checksum": true,
        "partial": true,
        "bwlimit": 1000,
        "itemize_changes": true,
        "stats": true
    }
}

```

**Notes:**

-   **`remote_host`**: IP address of **Server 254**.
-   **`ssh_key_path`**: Path to the SSH private key for authentication.

### **4.3. Python Synchronization Script**

**Path:** `/opt/opennms/rrd_monitoring/rrdPyScript.py`

**Content:**

```python
#!/usr/bin/env python3

import os
import json
import subprocess
import logging
import time
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from threading import Lock, Timer

# ---------------------------
# Load Configuration
# ---------------------------

CONFIG_FILE = "/opt/opennms/rrd_monitoring/config_rrdPyScript.json"

def load_config(config_path):
    """Load configuration from a JSON file."""
    if not os.path.exists(config_path):
        raise FileNotFoundError(f"Configuration file {config_path} does not exist.")
    with open(config_path, 'r') as f:
        try:
            config = json.load(f)
            return config
        except json.JSONDecodeError as e:
            raise ValueError(f"Error parsing JSON configuration: {e}")

config = load_config(CONFIG_FILE)

# ---------------------------
# Configuration Variables
# ---------------------------

SOURCE_DIR = config.get("source_dir", "/opt/opennms/rrd_data_move/")
STATE_FILE = config.get("state_file", "/opt/opennms/rrd_monitoring/rrdPyScript.json")

REMOTE_SYNC = config.get("remote_sync", {})
REMOTE_USER = REMOTE_SYNC.get("remote_user", "root")
REMOTE_HOST = REMOTE_SYNC.get("remote_host", "192.168.29.254")
REMOTE_DEST_DIR = REMOTE_SYNC.get("remote_dest_dir", "/opt/opennms/rrd_data_move/")
SSH_KEY_PATH = REMOTE_SYNC.get("ssh_key_path", "/root/.ssh/id_rsa_rrd_sync_primary")

LOGGING_CONFIG = config.get("logging", {})
LOG_FILE = LOGGING_CONFIG.get("log_file", "/opt/opennms/rrd_monitoring/rrdPyScript.log")
LOG_LEVEL = LOGGING_CONFIG.get("log_level", "INFO").upper()

DEBOUNCE_TIME = config.get("debounce_time", 5)  # seconds

MONITORED_EXTENSIONS = config.get("monitored_extensions", [".rrd", ".txt", ".properties"])

RSYNC_OPTIONS = config.get("rsync_options", {})
RSYNC_ARCHIVE = RSYNC_OPTIONS.get("archive", True)
RSYNC_COMPRESS = RSYNC_OPTIONS.get("compress", True)
RSYNC_VERBOSE = RSYNC_OPTIONS.get("verbose", True)
RSYNC_UPDATE = RSYNC_OPTIONS.get("update", True)
RSYNC_CHECKSUM = RSYNC_OPTIONS.get("checksum", True)
RSYNC_PARTIAL = RSYNC_OPTIONS.get("partial", True)
RSYNC_BWLIMIT = RSYNC_OPTIONS.get("bwlimit", 1000)
RSYNC_ITEMIZE = RSYNC_OPTIONS.get("itemize_changes", True)
RSYNC_STATS = RSYNC_OPTIONS.get("stats", True)

# ---------------------------
# Setup Logging
# ---------------------------

LOG_LEVEL_MAPPING = {
    "DEBUG": logging.DEBUG,
    "INFO": logging.INFO,
    "WARNING": logging.WARNING,
    "ERROR": logging.ERROR,
    "CRITICAL": logging.CRITICAL
}

logging.basicConfig(
    filename=LOG_FILE,
    level=LOG_LEVEL_MAPPING.get(LOG_LEVEL, logging.INFO),
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------
# Utility Functions
# ---------------------------

def load_state():
    """Load the synchronization state from a JSON file."""
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, 'r') as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                logging.error("State file is corrupted. Starting fresh.")
                return {}
    return {}

def save_state(state):
    """Save the synchronization state to a JSON file."""
    with open(STATE_FILE, 'w') as f:
        json.dump(state, f, indent=4)

# ---------------------------
# File System Event Handler
# ---------------------------

class RRDHandler(FileSystemEventHandler):
    """Handles filesystem events for synchronization."""
    def __init__(self, state):
        super().__init__()
        self.state = state
        self.monitored_extensions = MONITORED_EXTENSIONS
        self.lock = Lock()
        self.debounce_timers = {}

    def on_created(self, event):
        """Handle file creation events."""
        if not event.is_directory and self._is_monitored(event.src_path):
            self._debounced_sync(event.src_path, event_type="created")

    def on_modified(self, event):
        """Handle file modification events."""
        if not event.is_directory and self._is_monitored(event.src_path):
            self._debounced_sync(event.src_path, event_type="modified")

    def on_deleted(self, event):
        """Handle file deletion events."""
        if not event.is_directory and self._is_monitored(event.src_path):
            self._debounced_delete(event.src_path)

    def _is_monitored(self, path):
        """Check if the file has a monitored extension."""
        return any(path.endswith(ext) for ext in self.monitored_extensions)

    def _debounced_sync(self, src_path, event_type):
        """Debounce synchronization to prevent multiple triggers."""
        with self.lock:
            if src_path in self.debounce_timers:
                self.debounce_timers[src_path].cancel()
                logging.debug(f"Debounce timer canceled for {src_path}")
            timer = Timer(DEBOUNCE_TIME, self.sync_file, [src_path, event_type])
            self.debounce_timers[src_path] = timer
            timer.start()
            logging.debug(f"Debounce timer set for {src_path} with {DEBOUNCE_TIME} seconds")

    def _debounced_delete(self, src_path):
        """Debounce deletion to prevent multiple triggers."""
        with self.lock:
            if src_path in self.debounce_timers:
                self.debounce_timers[src_path].cancel()
                logging.debug(f"Debounce timer canceled for {src_path}")
            timer = Timer(DEBOUNCE_TIME, self.delete_file, [src_path])
            self.debounce_timers[src_path] = timer
            timer.start()
            logging.debug(f"Debounce timer set for deletion of {src_path} with {DEBOUNCE_TIME} seconds")

    def sync_file(self, src_path, event_type):
        """Synchronize a single file to the remote server while preserving directory structure."""
        relative_path = os.path.relpath(src_path, SOURCE_DIR)
        try:
            src_stat = os.stat(src_path)
            src_mtime = src_stat.st_mtime

            # Check if the file needs to be copied
            if (relative_path not in self.state) or (self.state[relative_path] < src_mtime):
                logging.info(f"Syncing {relative_path} due to {event_type} event.")

                # Build rsync command dynamically based on configuration
                rsync_command = ["rsync"]

                if RSYNC_ARCHIVE:
                    rsync_command.append("-a")
                if RSYNC_COMPRESS:
                    rsync_command.append("-z")
                if RSYNC_VERBOSE:
                    rsync_command.append("-v")
                if RSYNC_UPDATE:
                    rsync_command.append("--update")
                if RSYNC_CHECKSUM:
                    rsync_command.append("--checksum")
                if RSYNC_PARTIAL:
                    rsync_command.append("--partial")
                if RSYNC_BWLIMIT:
                    rsync_command.extend([f"--bwlimit={RSYNC_BWLIMIT}"])
                if RSYNC_ITEMIZE:
                    rsync_command.append("--itemize-changes")
                if RSYNC_STATS:
                    rsync_command.append("--stats")

                # Include the relative path in the destination to preserve directory structure
                destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
                rsync_command.extend([
                    "-e", f"ssh -i {SSH_KEY_PATH}",
                    src_path,
                    f"{REMOTE_USER}@{REMOTE_HOST}:{destination_path}"
                ])

                # Ensure that the destination directory exists
                destination_dir = os.path.dirname(destination_path)
                mkdir_command = [
                    "ssh",
                    "-i", SSH_KEY_PATH,
                    f"{REMOTE_USER}@{REMOTE_HOST}",
                    f"mkdir -p {destination_dir}"
                ]
                mkdir_result = subprocess.run(mkdir_command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
                if mkdir_result.returncode != 0:
                    logging.error(f"Failed to create directory {destination_dir} on remote host. Error: {mkdir_result.stderr.strip()}")
                    return  # Exit sync_file early due to failure

                # Execute rsync command
                result = subprocess.run(
                    rsync_command,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True
                )

                # Log rsync outputs
                if result.stdout:
                    logging.info(f"rsync output for {relative_path}:\n{result.stdout.strip()}")
                if result.stderr:
                    logging.warning(f"rsync warning/error for {relative_path}:\n{result.stderr.strip()}")

                if result.returncode == 0:
                    logging.info(f"Synced {relative_path} successfully.")
                    # Update state
                    self.state[relative_path] = src_mtime
                    save_state(self.state)
                else:
                    logging.error(f"Failed to sync {relative_path}. Error: {result.stderr.strip()}")
            else:
                logging.info(f"No changes detected for {relative_path}. Skipping.")
        except FileNotFoundError:
            logging.error(f"File {relative_path} not found for syncing.")
        except Exception as e:
            logging.error(f"Error syncing {relative_path}: {e}")

    def delete_file(self, src_path):
        """Delete a file from the remote server."""
        relative_path = os.path.relpath(src_path, SOURCE_DIR)
        try:
            logging.info(f"Deleting {relative_path} from destination...")

            # Build rsync delete command dynamically based on configuration
            rsync_delete_command = ["rsync"]

            if RSYNC_ARCHIVE:
                rsync_delete_command.append("-a")
            if RSYNC_COMPRESS:
                rsync_delete_command.append("-z")
            if RSYNC_VERBOSE:
                rsync_delete_command.append("-v")
            if RSYNC_UPDATE:
                rsync_delete_command.append("--update")
            if RSYNC_CHECKSUM:
                rsync_delete_command.append("--checksum")
            if RSYNC_PARTIAL:
                rsync_delete_command.append("--partial")
            if RSYNC_BWLIMIT:
                rsync_delete_command.extend([f"--bwlimit={RSYNC_BWLIMIT}"])
            if RSYNC_ITEMIZE:
                rsync_delete_command.append("--itemize-changes")
            if RSYNC_STATS:
                rsync_delete_command.append("--stats")
            if "--delete" not in rsync_delete_command:
                rsync_delete_command.append("--delete")

            rsync_delete_command.extend([
                "-e", f"ssh -i {SSH_KEY_PATH}",
                SOURCE_DIR,
                f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}/"
            ])

            # Execute rsync delete command
            result = subprocess.run(
                rsync_delete_command,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True
            )

            # Log rsync outputs
            if result.stdout:
                logging.info(f"rsync output for deletion:\n{result.stdout.strip()}")
            if result.stderr:
                logging.warning(f"rsync warning/error for deletion:\n{result.stderr.strip()}")

            if result.returncode == 0:
                logging.info(f"Deleted {relative_path} from destination successfully.")
                # Remove from state
                if relative_path in self.state:
                    del self.state[relative_path]
                    save_state(self.state)
            else:
                logging.error(f"Failed to delete {relative_path} from destination. Error: {result.stderr.strip()}")
        except Exception as e:
            logging.error(f"Error deleting {relative_path}: {e}")

# ---------------------------
# Initial Synchronization
# ---------------------------

def initial_sync(handler):
    """Perform initial synchronization of existing files."""
    logging.info("Starting initial synchronization of existing files...")
    for root, dirs, files in os.walk(SOURCE_DIR):
        for file in files:
            if any(file.endswith(ext) for ext in handler.monitored_extensions):
                src_path = os.path.join(root, file)
                handler.sync_file(src_path, event_type="initial sync")
    logging.info("Initial synchronization completed.")

# ---------------------------
# Main Execution
# ---------------------------

def main():
    logging.info("Starting RRD synchronization script.")

    # Load synchronization state
    state = load_state()

    # Create event handler
    event_handler = RRDHandler(state)

    # Perform initial synchronization
    initial_sync(event_handler)

    # Set up observer
    observer = Observer()
    observer.schedule(event_handler, SOURCE_DIR, recursive=True)
    observer.start()
    logging.info("Started monitoring for file changes.")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        logging.info("Stopping RRD synchronization script.")
    observer.join()

if __name__ == "__main__":
    main()

```

**Notes:**

-   **Debounce Time**: Prevents multiple rapid syncs by waiting for 5 seconds (`debounce_time`) after a file event before syncing.
-   **Rsync Options**: Configured for efficient and secure synchronization, preserving file attributes and minimizing bandwidth usage.

### **4.4. Systemd Service Setup**

**Path:** `/etc/systemd/system/rrdPyScript.service`

**Content:**

```ini
[Unit]
Description=RRD File-Level Synchronization Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/opennms/rrd_monitoring/rrdPyScript.py
Restart=on-failure
RestartSec=30
User=root
Environment=PYTHONUNBUFFERED=1


[Install]
WantedBy=multi-user.target

```

**Actions:**

1.  **Create the Service File:**
    
    ```bash
    sudo nano /etc/systemd/system/rrdPyScript.service
    
    ```
    
2.  **Paste the Above Content and Save:**
    
    -   **In `nano`:** Press `CTRL + O` to save, `ENTER` to confirm, and `CTRL + X` to exit.
3.  **Reload `systemd` and Enable the Service:**
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable rrdPyScript.service
    
    ```
    
4.  **Start the Service:**
    
    ```bash
    sudo systemctl start rrdPyScript.service
    
    ```
    
5.  **Verify Service Status:**
    
    ```bash
    sudo systemctl status rrdPyScript.service
    
    ```
    
    **Expected Output:**
    
    ```
    ● rrdPyScript.service - RRD File-Level Synchronization Service
         Loaded: loaded (/etc/systemd/system/rrdPyScript.service; enabled; vendor preset: enabled)
         Active: active (running) since Thu 2024-12-26 22:50:00 UTC; 1min ago
       Main PID: 12346 (python3)
          Tasks: 1 (limit: 4915)
         Memory: 50.0M
            CPU: 0.10s
         CGroup: /system.slice/rrdPyScript.service
                 └─12346 /usr/bin/python3 /opt/opennms/rrd_monitoring/rrdPyScript.py
    
    ```
    

### **4.5. Initial Synchronization Steps**

1.  **Ensure SSH Key Authentication:**
    
    -   **Generate SSH Keys on Server 106 (If Not Already Done):**
        
        ```bash
        ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa_rrd_sync_primary -N ""
        
        ```
        
    -   **Copy Public Key to Server 254:**
        
        ```bash
        ssh-copy-id -i /root/.ssh/id_rsa_rrd_sync_primary.pub root@192.168.29.254
        
        ```
        
2.  **Verify SSH Connectivity:**
    
    -   **From Server 106 to Server 254:**
        
        ```bash
        ssh -i /root/.ssh/id_rsa_rrd_sync_primary root@192.168.29.254 "echo 'SSH Connection Successful'"
        
        ```
        
    -   **Expected Output:**
        
        ```
        SSH Connection Successful
        
        ```
        
3.  **Populate `rrd_data_move` Directory on Server 106:**
    
    -   **Create Sample Files:**
        
        ```bash
        sudo touch /opt/opennms/rrd_data_move/sample1.rrd
        sudo touch /opt/opennms/rrd_data_move/sample2.txt
        
        ```
        
4.  **Monitor Synchronization Logs on Server 254:**
    
    ```bash
    sudo tail -f /opt/opennms/rrd_monitoring/rrdPyScript.log
    
    ```
    
    **Look For:**
    
    -   **Startup Messages:** Confirmation that the script has started.
    -   **Sync Attempts:** Logs indicating synchronization of `sample1.rrd` and `sample2.txt`.
    -   **Success Messages:** Indications of successful syncs.
5.  **Verify Files on Server 254:**
    
    ```bash
    ssh root@192.168.29.254 "ls /opt/opennms/rrd_data_move/"
    
    ```
    
    **Expected Output:**
    
    ```
    sample1.rrd
    sample2.txt
    
    ```
    

----------

## **5. Failover Synchronization: Server 254 to Server 106**

In the event of **Server 106** failure, you need to reverse the synchronization direction, making **Server 254** the **Primary Source** and **Server 106** the **Destination** once it's back online.

### **5.1. Directory Structure**

After renaming (removing "secondary"):

**Server 254 (Now Primary Source):**

```bash
/opt/opennms/
├── rrd_data_move/
│   └── [Existing and New Files]
└── rrd_monitoring/
    ├── config_rrdPyScript.json
    ├── rrdPyScript.py
    ├── rrdPyScript.json
    └── rrdPyScript.log

```

**Server 106 (Now Destination):**

```bash
/opt/opennms/
├── rrd_data_move/
│   └── [Existing Files from Initial Sync]
└── rrd_monitoring/
    ├── config_rrdPyScript.json
    ├── rrdPyScript.py
    ├── rrdPyScript.json
    └── rrdPyScript.log

```

### **5.2. Configuration File**

**Path:** `/opt/opennms/rrd_monitoring/config_rrdPyScript.json`

**Content:**

```json
{
    "source_dir": "/opt/opennms/rrd_data_move/",
    "state_file": "/opt/opennms/rrd_monitoring/rrdPyScript.json",
    "remote_sync": {
        "remote_user": "root",
        "remote_host": "192.168.29.106",
        "remote_dest_dir": "/opt/opennms/rrd_data_move/",
        "ssh_key_path": "/root/.ssh/id_rsa_rrd_sync_secondary"
    },
    "logging": {
        "log_file": "/opt/opennms/rrd_monitoring/rrdPyScript.log",
        "log_level": "INFO"
    },
    "debounce_time": 5,
    "monitored_extensions": [
        ".rrd",
        ".txt",
        ".properties",
        ".meta"
    ],
    "rsync_options": {
        "archive": true,
        "compress": true,
        "verbose": true,
        "update": true,
        "checksum": true,
        "partial": true,
        "bwlimit": 1000,
        "itemize_changes": true,
        "stats": true
    }
}

```

**Notes:**

-   **`remote_host`**: Now points back to **Server 106**.
-   **`ssh_key_path`**: Ensure that **Server 254** has the SSH key (`id_rsa_rrd_sync_primary`) that can authenticate with **Server 106**.

### **5.3. Python Synchronization Script**

**Path:** `/opt/opennms/rrd_monitoring/rrdPyScript.py`

**Content:**  
The same script as in **Initial Synchronization**, no changes required.

**Notes:**

-   Ensure that the script is correctly referencing the updated `state.json` and configuration.
-   The script should now monitor **Server 254's** `rrd_data_move` directory and sync changes to **Server 106**.

### **5.4. Systemd Service Setup**

**Path:** `/etc/systemd/system/rrdPyScript.service`

**Content:**  
Same as in **Initial Synchronization**.

**Notes:**

-   The service remains the same; no need to create a new service.
-   Ensure that the service file points to the updated script and configuration.

### **5.5. Failover Steps**

**Objective:**  
Promote **Server 254** to primary and reverse synchronization direction.

**Actions:**

1.  **Backup Existing `state.json` Files:**
    
    -   **On Server 254:**
        
        ```bash
        sudo cp /opt/opennms/rrd_monitoring/rrdPyScript.json /opt/opennms/rrd_monitoring/rrdPyScript.json.bak
        
        ```
        
    -   **On Server 106:**
        
        ```bash
        sudo cp /opt/opennms/rrd_monitoring/rrdPyScript.json /opt/opennms/rrd_monitoring/rrdPyScript.json.bak
        
        ```
        
2.  **Copy `state.json` from Server 106 to Server 254:**
    
    -   **On Server 254:**
        
        ```bash
        scp root@192.168.29.106:/opt/opennms/rrd_monitoring/rrdPyScript.json /opt/opennms/rrd_monitoring/rrdPyScript.json
        
        ```
        
    
    **Explanation:**
    
    -   This copies the synchronization state, ensuring that **Server 254** recognizes the 100 existing files and only syncs new ones.
3.  **Update Configuration File on Server 254 (if necessary):**
    
    -   **Open Configuration File:**
        
        ```bash
        sudo nano /opt/opennms/rrd_monitoring/config_rrdPyScript.json
        
        ```
        
    -   **Ensure Parameters are Correct:**
        
        -   **`remote_host`**: Should be **Server 106**'s IP.
        -   **`source_dir`**: `/opt/opennms/rrd_data_move/` (where new files are created on **Server 254**).
    -   **Save and Exit:**
        
        -   **In `nano`:** Press `CTRL + O` to save, `ENTER` to confirm, and `CTRL + X` to exit.
4.  **Restart the Synchronization Service on Server 254:**
    
    ```bash
    sudo systemctl restart rrdPyScript.service
    
    ```
    
5.  **Verify Service Status:**
    
    ```bash
    sudo systemctl status rrdPyScript.service
    
    ```
    
    **Expected Output:**
    
    ```
    ● rrdPyScript.service - RRD File-Level Synchronization Service
         Loaded: loaded (/etc/systemd/system/rrdPyScript.service; enabled; vendor preset: enabled)
         Active: active (running) since Thu 2024-12-26 23:10:00 UTC; 1min ago
       Main PID: 12346 (python3)
          Tasks: 1 (limit: 4915)
         Memory: 50.0M
            CPU: 0.10s
         CGroup: /system.slice/rrdPyScript.service
                 └─12346 /usr/bin/python3 /opt/opennms/rrd_monitoring/rrdPyScript.py
    
    ```
    
6.  **Ensure SSH Connectivity from Server 254 to Server 106:**
    
    ```bash
    ssh -i /root/.ssh/id_rsa_rrd_sync_primary root@192.168.29.106 "echo 'SSH Connection Successful'"
    
    ```
    
    **Expected Output:**
    
    ```
    SSH Connection Successful
    
    ```
    
7.  **Monitor Synchronization Logs on Server 254:**
    
    ```bash
    sudo tail -f /opt/opennms/rrd_monitoring/rrdPyScript.log
    
    ```
    
    **Look For:**
    
    -   **Startup Messages:** Confirmation that the script has started.
    -   **Sync Attempts:** Logs indicating synchronization of new files.
    -   **Success Messages:** Indications of successful syncs.

----------

## **6. Testing the Synchronization**

To ensure that both synchronization directions work seamlessly, perform the following tests:

### **6.1. Test Initial Synchronization (106 ➔ 254)**

1.  **On Server 106:**
    
    ```bash
    sudo touch /opt/opennms/rrd_data_move/test_initial_sync.txt
    sudo echo "Initial sync test" > /opt/opennms/rrd_data_move/test_initial_sync.txt
    
    ```
    
2.  **Monitor Logs on Server 254:**
    
    ```bash
    sudo tail -f /opt/opennms/rrd_monitoring/rrdPyScript.log
    
    ```
    
    **Look For:**
    
    ```
    Syncing test_initial_sync.txt due to created event.
    Synced test_initial_sync.txt successfully.
    
    ```
    
3.  **Verify on Server 254:**
    
    ```bash
    ssh root@192.168.29.254 "ls /opt/opennms/rrd_data_move/"
    
    ```
    
    **Expected Output:**
    
    ```
    test_initial_sync.txt
    
    ```
    

### **6.2. Test Failover Synchronization (254 ➔ 106)**

1.  **Simulate Server 106 Failure:**
    
    -   **Stop the Synchronization Service on Server 106:**
        
        ```bash
        ssh root@192.168.29.106 "sudo systemctl stop rrdPyScript.service"
        
        ```
        
2.  **On Server 254:**
    
    ```bash
    sudo touch /opt/opennms/rrd_data_move/test_failover_sync.txt
    sudo echo "Failover sync test" > /opt/opennms/rrd_data_move/test_failover_sync.txt
    
    ```
    
3.  **Monitor Logs on Server 254:**
    
    ```bash
    sudo tail -f /opt/opennms/rrd_monitoring/rrdPyScript.log
    
    ```
    
    **Look For:**
    
    ```
    Syncing test_failover_sync.txt due to created event.
    Synced test_failover_sync.txt successfully.
    
    ```
    
4.  **Verify on Server 106:**
    
    ```bash
    ssh root@192.168.29.106 "ls /opt/opennms/rrd_data_move/"
    
    ```
    
    **Expected Output:**
    
    ```
    test_initial_sync.txt
    test_failover_sync.txt
    
    ```
    
5.  **Restore Server 106:**
    
    -   **Start the Synchronization Service on Server 106:**
        
        ```bash
        ssh root@192.168.29.106 "sudo systemctl start rrdPyScript.service"
        
        ```
        

----------

## **7. Troubleshooting**

If you encounter issues during synchronization, follow these steps:

### **7.1. Check Service Status**

-   **On Both Servers:**
    
    ```bash
    sudo systemctl status rrdPyScript.service
    
    ```
    
    **Ensure the service is active and running without errors.**
    

### **7.2. Review Synchronization Logs**

-   **On Both Servers:**
    
    ```bash
    sudo tail -f /opt/opennms/rrd_monitoring/rrdPyScript.log
    
    ```
    
    **Look For:**
    
    -   **Error Messages:** Indicate issues with SSH, permissions, or rsync.
    -   **Sync Attempts:** Confirmation that the script is attempting to sync files.

### **7.3. Verify SSH Key Authentication**

-   **From Source to Destination:**
    
    ```bash
    ssh -i /root/.ssh/id_rsa_rrd_sync_primary root@192.168.29.254 "echo 'SSH Connection Successful'"
    
    ```
    
    **Ensure it returns `SSH Connection Successful`.**
    
-   **From Server 254 to Server 106 (During Failover):**
    
    ```bash
    ssh -i /root/.ssh/id_rsa_rrd_sync_primary root@192.168.29.106 "echo 'SSH Connection Successful'"
    
    ```
    

### **7.4. Validate `state.json` Integrity**

-   **On Both Servers:**
    
    ```bash
    cat /opt/opennms/rrd_monitoring/rrdPyScript.json
    
    ```
    
    **Ensure it contains accurate entries for synchronized files.**
    

### **7.5. Test Rsync Manually**

-   **From Source Server:**
    
    ```bash
    rsync -avz -e "ssh -i /root/.ssh/id_rsa_rrd_sync_primary" /opt/opennms/rrd_data_move/test_manual_sync.txt root@192.168.29.254:/opt/opennms/rrd_data_move/
    
    ```
    
-   **Verify on Destination Server:**
    
    ```bash
    ssh root@192.168.29.254 "ls /opt/opennms/rrd_data_move/"
    
    ```
    

### **7.6. Check Permissions and Ownership**

-   **Ensure that the synchronization script and directories have appropriate permissions:**
    
    ```bash
    sudo chmod -R 750 /opt/opennms/rrd_monitoring/
    sudo chown -R root:root /opt/opennms/rrd_monitoring/
    sudo chmod 600 /root/.ssh/id_rsa_rrd_sync_primary
    sudo chmod 700 /root/.ssh
    
    ```
    

### **7.7. Restart Services**

-   **On Both Servers:**
    
    ```bash
    sudo systemctl restart rrdPyScript.service
    
    ```
    

### **7.8. Validate Directory Paths**

-   **Ensure that `source_dir` and `remote_dest_dir` in configuration files correctly point to the intended directories.**

