# moddable-espnow

`moddable-espnow` is a Moddable SDK module for ESP-NOW communication on ESP32 devices. It supports broadcast and unicast messaging, peer management, and encrypted unicast communication.

## Modules

| Module            | Purpose                                                                               | Manifest                         |
| ----------------- | ------------------------------------------------------------------------------------- | -------------------------------- |
| `espnow`          | Promise-based sending, bounded send queue, and automatic Wi-Fi initialization         | `manifest.json`                  |
| `espnow/core`     | Low-level packet reads and writes with MAC send results                               | `modules/core/manifest.json`     |
| `espnow/reliable` | Optional application acknowledgments, retries, and duplicate suppression for commands | `modules/reliable/manifest.json` |

Use `espnow` for most applications. Use `espnow/core` when the application needs to manage Wi-Fi and send scheduling directly. These are project-specific APIs inspired by Moddable's I/O patterns.

## Features

- Broadcast and unicast transmission using Byte Buffers
- Receive events with sender MAC, destination, RSSI, and channel
- Promise-based send queue with configurable limits and timeout
- Replacement of queued updates with `sendLatest()`
- Peer registration and 16-byte PMK/LMK configuration
- Explicit recovery after a send completion timeout
- TypeScript implementations and exported public types

## Setup

### 1) Prepare the SDK

Install the [Moddable SDK](https://github.com/Moddable-OpenSource/moddable) and the ESP-IDF version required by your SDK checkout. Configure `MODDABLE`, `IDF_PATH`, and the SDK build tools before building for your ESP32 target.

### 2) Include the module in your manifest

Add the module to your application's `manifest.json`:

```json
{
  "include": [
    "$(MODDABLE)/examples/manifest_base.json",
    {
      "git": "https://github.com/stc1988/moddable-espnow.git"
    }
  ]
}
```

The repository's root `manifest.json` includes the default module, so the Git URL alone is sufficient. For a local checkout, replace the Git include object with the path to this repository's root directory, relative to your application's manifest.

The default manifest includes both the ESP-NOW core and Wi-Fi interface. To use only the core, select `modules/core/manifest.json` and initialize Wi-Fi in your application. To add the reliable command layer, select `modules/reliable/manifest.json`, which also includes the default module.

### 3) Choose a channel

Set both devices to the same Wi-Fi channel. The default wrapper initializes Wi-Fi and uses channel 1 when it manages the interface. An existing access-point connection constrains the channel to that of the access point.

## Minimal Usage

```js
import ESPNow from "espnow";
import Timer from "timer";

const radio = new ESPNow({
  channel: 1,
  onReceive({ peerId, data, rssi }) {
    trace(`${peerId}: ${data.byteLength} bytes, ${rssi} dBm\n`);
  },
});

trace(`ESP-NOW address: ${radio.id}\n`);

Timer.repeat(() => {
  radio
    .send(undefined, ArrayBuffer.fromString("hello"))
    .catch((error) => trace(`${error.message}\n`));
}, 1000);
```

`send(undefined, data)` broadcasts to nearby devices on the same channel. For unicast, register the recipient before sending:

```js
const peerId = "AABBCCDDEEFF"; // Replace with the recipient's MAC address.
radio.addPeer(peerId);
await radio.send(peerId, Uint8Array.of(1, 2, 3));
```

For encrypted unicast, both devices must register each other with the same 16-byte LMK. Broadcast packets cannot be encrypted.

## Event Model

### `onReceive(event)`

Called when a packet arrives. The callback's `this` is the `ESPNow` instance.

| Field         | Description                          |
| ------------- | ------------------------------------ |
| `data`        | Received `ArrayBuffer`               |
| `peerId`      | Sender MAC address                   |
| `destination` | Destination MAC address              |
| `broadcast`   | Whether the destination is broadcast |
| `rssi`        | Received signal strength in dBm      |
| `channel`     | Receive channel                      |

MAC addresses returned by the API use uppercase hexadecimal without separators.

### `send(peerId, data)`

Returns a Promise that resolves on MAC-layer send success. This does not confirm that the remote application received or processed the packet. Use an application reply or `espnow/reliable` when that confirmation is required.

### `onError(error)`

Called when the send completion notification times out. The radio enters a faulted state and rejects pending sends. Call `recover()` to reinitialize the core and restore peers and keys. Failed sends are not replayed automatically.

## API

### ESPNow

| API                                   | Description                                                                                                             |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `new ESPNow(options = {})`            | Create a radio; options include `channel`, `manageWiFi`, `queueLimit`, `sendTimeout`, `pmk`, `onReceive`, and `onError` |
| `id` / `info`                         | Local MAC address and radio information                                                                                 |
| `pending` / `faulted`                 | Number of unfinished sends and fault state                                                                              |
| `send(peerId, data)`                  | Copy and queue a packet for broadcast or unicast                                                                        |
| `sendLatest(peerId, data, key = "")`  | Replace a waiting update for the same recipient and key                                                                 |
| `addPeer(peerId, options = {})`       | Register or update a peer; options include `channel`, `interface`, and `lmk`                                            |
| `getPeer(peerId)` / `hasPeer(peerId)` | Inspect peer registration                                                                                               |
| `removePeer(peerId)`                  | Remove a peer                                                                                                           |
| `configure(options)`                  | Change channel or PMK while no sends are pending                                                                        |
| `recover()`                           | Restore a faulted radio without replaying failed sends                                                                  |
| `close()` / `[Symbol.dispose]()`      | Release resources and reject unfinished sends                                                                           |

The default send queue limit is 32 packets, including the active send. The default send completion timeout is 2000 ms.

Payloads may be `ArrayBuffer`, `SharedArrayBuffer`, `Uint8Array`, `Int8Array`, or `DataView`. The maximum length is available as `radio.info.maxDataLength`. Keep packets within 250 bytes when communicating with ESP-NOW v1 devices; supported v2 devices allow up to 1470 bytes. Peer capabilities are not detected automatically.

### ESPNowCore and ReliableESPNow

`espnow/core` provides `write()`, `read()`, `onReadable()`, and `onSent()` for direct packet handling. Only one packet may be in flight; wait for `onSent()` before the next write. The application owns Wi-Fi initialization and shutdown.

`espnow/reliable` adds a separate unicast command protocol with acknowledgments, retries, and in-memory duplicate suppression. Both ends must use that protocol. Duplicate suppression does not survive a receiver restart.

See the [module API guide](docs/README.md) for options, recovery behavior, and reliable command examples, and the [design and operation guide](docs/BEST_PRACTICES.md) for implementation constraints and application design.

## Examples

| Example                                                                                           | Description                                                               |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [broadcast/minimal](examples/broadcast/minimal)                                                   | Periodic broadcast with receive logging                                   |
| [broadcast/sender](examples/broadcast/sender) / [broadcast/receiver](examples/broadcast/receiver) | Piu display showing JSON PING/PONG messages and round-trip time           |
| [imu-sync](examples/imu-sync/README.md)                                                           | Synchronize accelerometer readings and ball position between two displays |
| [idf-peer](examples/idf-peer/README.md)                                                           | ESP-IDF-format peer discovery and encrypted unicast using the core API    |

The IMU example uses two ESP32 devices with 320×240 displays, with an IMU on the sender. From the repository root, build and flash each device separately:

```sh
(cd examples/imu-sync/sender && mcconfig -dl -m -p esp32/m5stack_cores3)
(cd examples/imu-sync/receiver && mcconfig -dl -m -p esp32/m5stack_cores3)
```

Connect only the device being flashed. These commands target M5Stack CoreS3; select the appropriate SDK target for other hardware. The `-dl` option opens the command-line debugger for logs.

## Development

Install development dependencies:

```sh
npm ci
```

Format, check formatting and lint, and check TypeScript types:

```sh
npm run format
npm run check
npm run typecheck
```

Run the module and IMU sample tests from the repository root:

```sh
PATH="$PWD/node_modules/.bin:$PATH" node --test tests/*.test.mjs examples/imu-sync/tests/*.test.mjs
```

Tests use host-side substitutes for the radio, Wi-Fi, timers, and display. The native callback tests also require a C compiler (`cc`) with AddressSanitizer and UndefinedBehaviorSanitizer support. Device builds and real radio communication must be checked on hardware.

## License

[The Unlicense](LICENSE).
