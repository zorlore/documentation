# ZorLore Bridge

ZorLore Bridge is the backend layer between the game server and persistent data.  
It provides the integration point for configuration, account/runtime data, and backend services.

## Download

Download Bridge releases from:

- `https://github.com/zorlore/zorlore-bridge/releases`

## `config.yaml` Explained

Path: `bridge/config.yaml`

### World and Mode

- `world`: World display name.
- `worldType`: `pvp`, `nonpvp`, or `pvpenforced`.
- `privateWorld`: Enables private-world behavior.
- `freePremium`: Enables free premium mode.
- `saveTime`: Server save/reboot time (`HHMM` or `HH:MM`).

### Game Endpoint

- `gameAddress`: Default game endpoint used for character-list routing when no override exists.
- `wsGamePort`: Game server WebSocket port.

### Bridge Endpoint

- `bridgeHost`: Bridge bind/announce host.
- `bridgePort`: Bridge RPC port (TCP).
- `bridgeWsPort`: Bridge WebSocket port.
- `bridgeSharedSecret`: Shared secret. Must match game server config.

### Storage

- `backend`: Backend type (currently sqlite).
- `dataPath`: Data directory for bridge-owned data.
- `sqlitePath`: SQLite database file path.
- `logPath`: Log output directory.

## Runtime Dependencies

### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y libcrypto3 libsqlite3-0 liblua5.4-0
```

### macOS

```bash
brew install openssl@3 sqlite lua
```

### Windows

Install MSYS2 runtime packages:

```powershell
pacman -S --needed mingw-w64-x86_64-openssl mingw-w64-x86_64-sqlite3 mingw-w64-x86_64-lua
```

Then either:

- add `C:\msys64\mingw64\bin` to `PATH`, or
- copy required DLLs next to `bin\bridge`.

## Run

From the extracted release folder:

```bash
./bin/bridge
```

On Windows:

```powershell
.\bin\bridge.exe
```

Start Bridge before the game server so RPC/world config calls are available at server startup.

