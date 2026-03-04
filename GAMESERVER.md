# ZorLore Game Server

The ZorLore game server is the core runtime of your MMORPG world.  
It executes gameplay logic, loads your world data, manages creatures and combat, and serves live game sessions.

## Download

Download the latest server builds from:

- `https://github.com/zorlore/zorlore-server/releases`

## How It Connects To Bridge

The game server depends on ZorLore Bridge for backend operations (account/world config queries, persistence, and integration flow).

These values in `gameserver/config.yaml` must match Bridge:

- `bridgeHost`
- `bridgePort`
- `bridgeWsPort`
- `bridgeSharedSecret`

If host/ports/secret do not match, startup or backend calls will fail.

## `config.yaml` Explained

Path: `gameserver/config.yaml`

### Bridge

- `bridgeHost`: IP/host where Bridge RPC is reachable.
- `bridgePort`: Bridge RPC port (TCP).
- `bridgeWsPort`: Bridge WebSocket port.
- `bridgeSharedSecret`: Shared secret used by server and bridge.
- `backendSqlitePath`: Local sqlite path used by the server-side backend flow.
- `freepremium`: Enables free premium behavior.

### Paths

- `binPath`: Binary path.
- `dataPath`: Data folder (world/assets data location).
- `logPath`: Log output folder.
- `savePath`: Runtime save/tmp output folder.
- `npcPath`: NPC script/data path.

### World

- `world`: World name.
- `state`: Server state (`public`, etc.).

### Save Window

- `saveTime`: Daily save/restart time (`HHMM` or `HH:MM`).

## Runtime Dependencies

### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y libssl3 libsqlite3-0 liblua5.4-0
```

### macOS

```bash
brew install openssl@3 sqlite lua
```

### Windows

Install MSYS2 and runtime packages:

```powershell
pacman -S --needed mingw-w64-x86_64-openssl mingw-w64-x86_64-sqlite3 mingw-w64-x86_64-lua
```

Then either:

- add `C:\msys64\mingw64\bin` to `PATH`, or
- copy required DLLs next to `bin\game`.

## Run

From the extracted release folder:

```bash
./bin/game
```

On Windows:

```powershell
.\bin\game.exe
```

Make sure Bridge is running first and both `config.yaml` files are aligned.

