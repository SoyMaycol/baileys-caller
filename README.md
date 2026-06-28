# baileys-caller

Place outbound WhatsApp voice calls from Node.js.

`baileys-caller` wraps WhatsApp Web's VoIP WASM stack, uses Baileys for authentication/signaling, and feeds audio through `ffmpeg` into the live RTP session.

> **Author:** ShellTear

## Status

- ✅ Outbound 1:1 voice calls
- ✅ Reuse an existing Baileys socket from a bot
- ✅ Reuse or auto-detect Baileys multi-file auth sessions
- ✅ Stream audio from MP3/WAV files, lavfi sources, or silence
- ✅ Receive remote audio as `Float32Array`
- ✅ Mute / unmute / hang up
- ❌ Group calls
- ❌ Video
- ❌ Inbound calls

## Requirements

- Node.js ≥ 20
- `ffmpeg` on `PATH` for non-silence audio sources
- Baileys installed in the host app (`@whiskeysockets/baileys` or the compatible `baileys` package your bot already uses)
- Network access to WhatsApp Web signaling and relay endpoints

## Install

This package is not published on npm. Pull it from git:

```bash
git clone https://github.com/SheIITear/baileys-caller
cd baileys-caller
npm install
npm run build
```

Or depend on the repo from another project:

```json
{
  "dependencies": {
    "baileys-caller": "git+https://github.com/SheIITear/baileys-caller.git",
    "@whiskeysockets/baileys": "^7.0.0-rc11"
  }
}
```

## Recommended bot usage: reuse the existing socket

If your bot already has a connected Baileys `sock`, pass it to `VoipClient`. Do **not** open a second socket with the same session directory; WhatsApp may replace the first connection and Baileys can later throw `Error: Connection Closed`.

```ts
import { VoipClient } from "baileys-caller";

const client = new VoipClient({
  sock,                         // the already-connected Baileys socket from your bot
  baileys: await import("baileys"), // or await import("@whiskeysockets/baileys")
  authDir: "./Sessions/Owner",  // optional when `sock` is provided; useful for clarity
});

await client.connect(); // initializes only the VoIP stack; it does not reconnect WhatsApp

const call = await client.call("12345678901", {
  audioSource: "./hello.mp3",
  durationMs: 120_000,
});

call.on("ringing", () => console.log("ringing"));
call.on("connected", () => console.log("connected"));
call.on("ended", (reason) => console.log("ended:", reason));

await call.waitForEnd();
client.disconnect(); // leaves the external bot socket open
```

### Example command handler

```ts
import { writeFile, unlink, mkdir } from "node:fs/promises";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";
import { VoipClient } from "baileys-caller";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
const sessionDir = join(__dirname, "../../Sessions/Owner");

export default {
  command: ["calling"],
  category: "owner",
  description: "Llama a un número usando un audio respondido.",
  isOwner: true,

  run: async ({ msg, sock, args }) => {
    let audioPath: string | null = null;
    let client: VoipClient | null = null;

    try {
      const number = args?.[0]?.replace(/\D/g, "");
      if (!number) return msg.reply("Uso:\n.calling <numero>\n\nResponde a un audio.");

      const quoted = msg.quoted;
      const isAudio = quoted?.message?.audioMessage || quoted?.msg?.mimetype?.startsWith("audio/");
      if (!isAudio) return msg.reply("Debes responder a un audio.");

      const audioBuffer = await quoted.download();
      if (!audioBuffer) return msg.reply("No pude descargar el audio.");

      await mkdir(join(__dirname, "../../tmp"), { recursive: true });
      audioPath = join(__dirname, "../../tmp", `call-${Date.now()}.mp3`);
      await writeFile(audioPath, audioBuffer);

      client = new VoipClient({
        sock,
        baileys: await import("baileys"),
        authDir: sessionDir,
      });

      await client.connect();
      await msg.reply(`Llamando a ${number}...`);

      const call = await client.call(number, { audioSource: audioPath });
      call.on("ringing", () => console.log("ringing"));
      call.on("connected", () => console.log("connected"));
      call.on("ended", (reason) => console.log("ended:", reason));

      await call.waitForEnd();
      return msg.reply("Llamada finalizada.");
    } finally {
      client?.disconnect();
      if (audioPath) await unlink(audioPath).catch(() => {});
    }
  },
};
```

## Standalone usage

If you are not inside an existing bot, let `VoipClient` create its own Baileys socket:

```ts
import { VoipClient } from "baileys-caller";

const client = new VoipClient({ authDir: "./auth" });
await client.connect(); // first run prints a QR for WhatsApp > Linked Devices

const call = await client.call("12345678901", { audioSource: "./hello.mp3" });
await call.waitForEnd();
client.disconnect(); // closes the socket created by VoipClient
```

You can also pass a session directory directly:

```ts
const client = new VoipClient("./Sessions/Owner");
```

Run the bundled example from a clone:

```bash
npx tsx examples/call.mts ./auth 12345678901 ./hello.mp3
```

## API

### `new VoipClient(options?)`

`options` can be either a config object or a string session path.

| Option | Type | Description |
|---|---:|---|
| `authDir` | `string?` | Baileys multi-file auth directory. When omitted, env vars and common local folders are auto-detected before falling back to `./auth`. |
| `sessionDir` | `string?` | Alias for `authDir`, useful when reusing an existing bot session folder. |
| `autoDetectAuthDir` | `boolean?` | Enables common-folder auto-detection when no directory is provided. Default: `true`. |
| `sock` | `any?` | Existing Baileys socket to reuse. When provided, `connect()` does not open a second WhatsApp connection. |
| `baileys` | `any?` | Optional imported Baileys module. Pass this when your bot imports `baileys` instead of `@whiskeysockets/baileys`. |

Auth directory resolution order when `sock` is not provided:

1. Constructor string: `new VoipClient("./session")`
2. `authDir` or `sessionDir`
3. `BAILEYS_AUTH_DIR`, `BAILEYS_SESSION_DIR`, `WHATSAPP_AUTH_DIR`
4. Existing local folders containing `creds.json`: `./auth`, `./session`, `./sessions`, `./baileys_auth_info`, `./auth_info_baileys`, `./database/baileys`
5. Fallback: `./auth`

### `client.connect(): Promise<void>`

Initializes Baileys signaling and the WhatsApp VoIP WASM stack.

- With `sock`: reuses your existing Baileys connection and only attaches VoIP listeners.
- Without `sock`: creates a Baileys socket using `authDir`; first run prints a QR code.

### `client.call(phoneNumber, opts?): Promise<ActiveCall>`

Places an outbound call. `phoneNumber` may contain punctuation; non-digits are stripped.

| Option | Type | Description |
|---|---:|---|
| `audioSource` | `string?` | Path to MP3/WAV, `lavfi:<filter>`, or `"silence"`. Default: `"silence"`. |
| `durationMs` | `number?` | Auto-hangup after N ms. Default: `120000`. |

### `client.disconnect(): void`

Releases VoIP resources and listeners.

- If `VoipClient` created the Baileys socket, it closes that socket.
- If you passed `sock`, it leaves your bot socket open.

### `ActiveCall`

Returned by `client.call()`. Extends `EventEmitter`.

#### Events

| Event | Payload | When |
|---|---:|---|
| `ringing` | — | Remote device is ringing. |
| `connected` | — | Call answered and media is flowing. |
| `audio` | `Float32Array` | 16 kHz mono PCM frame from the remote peer. |
| `ended` | `string` | Call ended (`hangup`, `timeout`, `remote_end`, `disconnect`, etc.). |
| `error` | `Error` | Fatal call error. |

#### Methods

- `call.end(reason = "hangup"): void` — hang up.
- `call.mute(muted: boolean): void` — toggle outgoing mute.
- `call.waitForEnd(): Promise<string>` — resolves with the final end reason.

#### Properties

- `call.callId: string`

## Troubleshooting

### `WARNING Conexión reemplazada — cerrá la otra sesión antes de reconectar`

You are opening more than one WhatsApp connection for the same Baileys session. In a bot command, pass the existing `sock` to `VoipClient` and do not create a second client with only `authDir`.

### `Error: Connection Closed` from Baileys `query` / `sendNode`

Usually the socket was replaced or closed. Check that your bot socket is connected before calling, pass `{ sock, baileys }`, and avoid calling `.end()` on the bot socket yourself. `client.disconnect()` will not close externally supplied sockets.

### `still waiting on run dependencies: loading-workers` or `pthread worker prewarm timed out`

These messages come from the WhatsApp Web WASM/Emscripten worker pool. They can appear during startup and are often non-fatal. Keep Node ≥ 20, avoid blocking the event loop while calling, and allow the first call a few seconds to initialize workers.

### No outgoing audio / immediate silence

Make sure `ffmpeg` is installed and the replied/downloaded audio file exists. If the audio file ends before the call ends, the feeder continues sending silence so the uplink stays alive.

## How it works

1. Baileys handles WhatsApp authentication, encryption, and signaling stanzas.
2. The WhatsApp Web VoIP WASM stack runs in-process to negotiate the call, encode/decode Opus, and manage RTP/SRTP.
3. A `worker_threads` pool mirrors the browser Web Worker environment expected by the WASM.
4. Outbound audio is decoded with `ffmpeg`, resampled to 16 kHz mono, fed into the WASM, and delivered to the relay.
5. Inbound audio is exposed as `Float32Array` chunks through the `audio` event.

## Auth state security

`authDir` / `sessionDir` contains Baileys session credentials. Treat it like a password: anyone with that folder can act as your linked WhatsApp device.

## WASM resources

The WASM binary and worker resources live under `assets/wasm/`:

- `whatsapp.wasm`
- `loader.js`
- `worker-modules.js`

To refresh fetched resources from a current WhatsApp Web session:

```bash
npm run fetch-wasm
```

The fetch script requires Chrome with remote debugging enabled on port `9222`.

## License

MIT © ShellTear
