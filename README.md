# Bluetooth File Transfer System - Technical Report

## Executive Summary

This report documents the development of a cross-platform Bluetooth file transfer system enabling seamless file exchange between Linux and Android devices. The system consists of a PyQt6-based application for Linux and a Kivy-based application for Android, utilizing RFCOMM Bluetooth protocol for reliable data transmission.

## System Architecture

### Overview
The system implements a peer-to-peer architecture where the Linux application initiates connections and transfers files to the Android application. Communication occurs over Bluetooth using the Serial Port Profile (SPP) with RFCOMM protocol.

### Key Components
- **Linux Application**: PyQt6 GUI application with Bluetooth device discovery and file management
- **Android Application**: Kivy-based application with file storage and export capabilities
- **Protocol**: Custom binary protocol for file transfers and directory synchronization
- **Storage**: Android app-specific internal storage with organized directory structure

## Linux Application

### Core Architecture

#### BluetoothScanner Class
```python
class BluetoothScanner(QObject):
```
- Implements PyQt6-based Bluetooth device discovery
- Uses `QBluetoothDeviceDiscoveryAgent` for scanning nearby devices
- Provides real-time device discovery with signal-based communication
- Automatically filters and displays discoverable Bluetooth devices

**Key Features:**
- Asynchronous device scanning with timeout handling
- Duplicate device filtering based on MAC addresses
- Thread-safe GUI updates using Qt signals
- Auto-stop functionality to prevent resource exhaustion

#### BluetoothClientGUI Class
```python
class BluetoothClientGUI(QMainWindow):
```
- Main application window with comprehensive file transfer interface
- Implements three-state connection management (disconnected/connecting/connected)
- Provides real-time logging with color-coded message types
- Supports directory synchronization operations

**GUI Components:**
- Device selection list with MAC address validation
- Connection status indicators with visual feedback
- File transfer progress monitoring
- Command input interface for remote application interaction

#### BluetoothFileClient Class
```python
class BluetoothFileClient:
```
- Handles all Bluetooth communication protocols
- Implements robust connection establishment with port scanning
- Manages directory synchronization with integrity checking
- Supports directory scanning and batch file operations

**Transfer Protocol:**

**Directory Synchronization**
- Directory scanning and file enumeration
- Batch file validation and filtering
- User confirmation dialogs with detailed file lists
- Sequential file transfer with error handling

### Connection Management

#### Port Discovery Process
```python
def find_and_connect(self, cancel_event=None):
    # Try common RFCOMM ports first (1, 2, 3, then scan others)
    port_order = [1, 2, 3] + list(range(4, 31))
```
- Prioritizes commonly used RFCOMM ports
- Implements cancellation support for user interruption
- Uses progressive timeout strategy
- Provides detailed connection attempt logging



### File Transfer Protocol

#### Directory Sync Protocol
```
1. Send "DIRSYNC" header (8 bytes)
2. Wait for "READY" response
3. Send directory name (4-byte length + name)
4. Send file count (4 bytes)
5. Wait for "CLEARED" response
6. For each file:
   - Send "NEWFILE" header
   - Wait for "READY"
   - Send metadata (filename, size, type)
   - Wait for "READY" or "REJECT"
   - Stream file data
   - Wait for "SAVED" confirmation
```

### Error Handling and Recovery
- Connection timeout management with progressive backoff
- File corruption detection through size verification
- User cancellation support during transfers
- Comprehensive error logging and user feedback

## Android Application

### Core Architecture

#### MyGridLayout Class
```python
class MyGridLayout(Widget):
```
- Main UI controller managing Bluetooth communication
- Implements file storage and organization systems
- Provides export functionality using Android Storage Access Framework
- Handles concurrent transfer operations with threading

#### Bluetooth Server Implementation
```python
def start_bluetooth_server(self):
    # Standard UUID for SPP (Serial Port Profile)
    MY_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
```
- Uses Android Bluetooth API through PyJNIus
- Implements RFCOMM socket listening
- Handles multiple connection attempts
- Provides automatic service discovery

### File Storage System

#### Internal Storage Management
```python
def initialize_storage(self):
    context = mActivity.getApplicationContext()
    files_dir = context.getFilesDir()
    self.app_storage_path = files_dir.getAbsolutePath()
```
- Utilizes Android app-specific internal storage
- Creates organized directory structure for received files
- Maintains in-memory file catalog for quick access
- Implements automatic cleanup and organization

#### Storage Features
- **Directory Organization**: Files organized by source directory name
- **Metadata Tracking**: File size, type, and path information
- **Export Capabilities**: Integration with Android Storage Access Framework
- **Storage Optimization**: Automatic file indexing and cataloging

### Transfer Handling

#### Protocol Implementation
The Android application implements the complementary side of the transfer protocols:

1. **Connection Acceptance**
   - Listens on RFCOMM socket
   - Handles connection confirmation handshake
   - Establishes bidirectional communication

2. **File Reception**
   - Validates incoming directory metadata
   - Implements streaming data reception for multiple files
   - Provides real-time progress feedback for batch operations
   - Ensures data integrity verification across all files

3. **Directory Synchronization**
   - Clears existing directory contents
   - Receives multiple files sequentially
   - Maintains transfer state consistency
   - Provides batch operation feedback

### User Interface Features

#### Command System
```python
def send_message(self):
    # Built-in commands: list, clear, show, export, shutdown
```
- Interactive command interface for file management
- Quick action buttons for common operations
- Real-time log display with color coding
- Status indicators for connection and transfer states

#### Export Functionality
```python
def create_file_with_extension(filetype, default_name="received_file"):
    intent = Intent(Intent.ACTION_CREATE_DOCUMENT)
```
- Integration with Android Storage Access Framework
- Support for multiple file types with appropriate MIME types
- User-friendly file selection and naming
- Direct export to user-accessible storage locations

### Threading and Concurrency

The Linux application implements a sophisticated multi-threading architecture to ensure responsive user interface while handling intensive Bluetooth operations:

#### Main Thread (GUI Thread)
- Handles all PyQt6 GUI updates and user interactions
- Processes Qt signals for thread-safe communication
- Manages device selection and connection status updates
- Responds to user input and button clicks

#### Bluetooth Scanner Thread
```python
self.bluetooth_scanner = BluetoothScanner(gui_callback=self)
```
- Runs asynchronous device discovery operations
- Uses Qt signals to communicate found devices back to main thread
- Implements timeout mechanisms to prevent indefinite scanning
- Automatically stops scanning when connection is established

#### Connection Thread
```python
self.connection_thread = threading.Thread(target=self.connect_to_server, daemon=True)
self.connection_thread.start()
```
- Handles the connection establishment process
- Performs port scanning without blocking the UI
- Supports user cancellation through threading events
- Uses `connection_cancelled` event for graceful termination

#### Command Receiver Thread
```python
receiver_thread = threading.Thread(target=self.client.receive_commands, daemon=True)
receiver_thread.start()
```
- Continuously listens for incoming commands from Android device
- Processes directory sync requests and file requests
- Implements transfer state management to pause command processing during active transfers
- Uses thread-safe logging through Qt signals

#### File Transfer Threads
```python
threading.Thread(target=self.send_file_worker, args=(filepath, ext), daemon=True).start()
threading.Thread(target=self.client.execute_directory_sync, args=(directory_path, files_to_send), daemon=True).start()
```
- Dedicated threads for file sending operations
- Prevents UI freezing during large file transfers
- Implements progress tracking and error handling
- Automatically re-enables UI controls upon completion

#### Thread Synchronization Mechanisms

**Transfer State Management**
```python
self.transfer_in_progress = False
self.transfer_lock = threading.Lock()
self.pause_commands = False
```
- Thread-safe transfer state tracking
- Prevents concurrent transfer operations
- Coordinates between command receiver and transfer threads

**Connection Cancellation**
```python
self.connection_cancelled = threading.Event()
```
- Allows user to cancel ongoing connection attempts
- Provides clean shutdown of connection threads
- Prevents resource leaks from abandoned connections

**Qt Signal-Slot Communication**
```python
log_signal = pyqtSignal(str)
status_signal = pyqtSignal(str)
connection_status_signal = pyqtSignal(str)
directory_sync_signal = pyqtSignal(str, list, list)
```
- Thread-safe communication between worker threads and GUI
- Ensures all UI updates occur on the main thread
- Prevents Qt threading violations and crashes

#### Thread Safety Considerations
- All GUI updates routed through Qt signals to main thread
- File operations performed on background threads
- Bluetooth operations isolated from UI thread
- Proper cleanup of daemon threads on application exit

## Build System and Deployment

### Virtual Environment Setup

Before building applications, it's essential to use virtual environments to isolate project dependencies. Many modern Linux distributions prohibit installing Python packages system-wide to prevent conflicts with system-managed packages. Virtual environments provide a clean, isolated space for project-specific dependencies.

#### Creating a Virtual Environment
```bash
# Create a new virtual environment
python3 -m venv bluetooth_transfer_env

# Activate the virtual environment
# On Linux/macOS:
source bluetooth_transfer_env/bin/activate

# On Windows:
bluetooth_transfer_env\Scripts\activate
```

#### Installing Project Dependencies
```bash
# With virtual environment activated
pip install --upgrade pip
pip install PyQt6
pip install kivy
pip install buildozer
pip install pyinstaller
```

#### Deactivating Virtual Environment
```bash
deactivate
```

Using virtual environments is particularly important on Linux distributions that enforce PEP 668 (externally managed environments), as they prevent direct installation of packages to the system Python environment. The virtual environment must remain activated throughout the build process for both Android APK and Linux executable generation.

### Android APK Building with Buildozer

Buildozer is a tool that automates the entire build process, downloading prerequisites like python-for-android, Android SDK, NDK, and other required tools. It manages a file named buildozer.spec in your application directory, describing your application requirements and settings such as title, icon, included modules etc.

#### Installation (Linux Ubuntu 20.04/22.04)

Buildozer is tested on Python 3.8 and above but may work on earlier versions, back to Python 3.3.

**System Dependencies Installation**
First, install the system dependencies:

```bash
sudo apt update
sudo apt install -y git zip unzip openjdk-17-jdk python3-pip autoconf libtool pkg-config zlib1g-dev libncurses5-dev libncursesw5-dev libtinfo5 cmake libffi-dev libssl-dev
```

**Python Dependencies**
Install Buildozer and required Python packages:

```bash
pip3 install --user --upgrade Cython==0.29.33 virtualenv
pip3 install --user --upgrade buildozer
```

**Environment Setup**
Add the following line at the end of your ~/.bashrc file:

```bash
export PATH=$PATH:~/.local/bin/
```

Then restart your terminal or run `source ~/.bashrc` to apply the changes.

#### Quickstart Build Process

**Project Initialization**
Create a buildozer.spec file with:

```bash
buildozer init
```

**Configuration**
Edit the buildozer.spec according to the Specifications. You should at least change the title, package.name and package.domain in the [app] section.

For the Bluetooth File Transfer application:
```ini
[app]
title = Bluetooth File Server
package.name = bluetoothfileserver
package.domain = org.example
requirements = python3,kivy,pyjnius
android.permissions = BLUETOOTH, BLUETOOTH_ADMIN, BLUETOOTH_CONNECT, BLUETOOTH_ADVERTISE, ACCESS_FINE_LOCATION, READ_EXTERNAL_STORAGE,WRITE_EXTERNAL_STORAGE

# Kivy version to use (must not exceed 1.9.1)
osx.kivy_version = 1.9.1
```

**Building the APK**
Start an Android/debug build with:

```bash
buildozer -v android debug
```

The first build will be slow, as it will download the Android SDK, NDK, and others tools needed for the compilation. Don't worry, those files will be saved in a global directory and will be shared across the different project you'll manage with Buildozer.

At the end, you should have an APK or AAB file in the bin/ directory.

This straightforward process automates the complex Android build pipeline, handling SDK/NDK downloads, dependency management, and APK generation through simple command-line operations.

### Linux Executable Building with PyInstaller

#### Build Process
1. **Installation**
   ```bash
   pip install pyinstaller
   ```

2. **Build Command**
   ```bash
   pyinstaller --hidden-import=PyQt6 --hidden-import=PyQt6.QtCore --hidden-import=PyQt6.QtWidgets --hidden-import=PyQt6.QtGui --hidden-import=PyQt6.QtBluetooth path/to/your/client.py -F
   ```

#### Build Optimizations
- **Hidden Imports**: Explicit inclusion of PyQt6.QtBluetooth module
- **Bundle Optimization**: Single-file executable generation
- **Resource Management**: Proper handling of Qt resources and plugins
- **Platform Compatibility**: Linux-specific Bluetooth library integration

## Conclusion

This Bluetooth file transfer system demonstrates effective cross-platform development using Python frameworks. The combination of PyQt6 for Linux and Kivy for Android provides robust, native-feeling applications on both platforms. The custom binary protocol ensures reliable data transfer, while the build systems (Buildozer and PyInstaller) enable easy distribution and deployment.

The project demonstrates practical solutions for real-world development challenges including responsive user interfaces during file transfers, reliable data transmission over Bluetooth, and simplified deployment across different platforms. The straightforward architecture makes it easy to modify and extend functionality as needed.