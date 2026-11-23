# Istruzioni per Claude: ricreare la mod `company-brain-progetti`

Questo file è per Claude Code. La persona te lo passa e ti chiede di installare la mod. Tu ricrei
la mod sul suo computer, file per file, la installi, la configuri e la verifichi. Segui i passi
nell'ordine. Ogni passo dice come controllarlo e cosa fare se non va.

## Cosa fa la mod

Collega Claude Code a un server Company Brain self-hosted. Per ogni
progetto (repository git, o cartella con un file come `package.json`) usa un vault di progetto:

- un server MCP stdio (`server/tools.mjs`) che fa da proxy verso `POST /mcp` del server e dà a
  Claude gli strumenti `brain_*`;
- gli hook (`hooks/register.tsx`) che a ogni commit allineano nel vault i file elencati in
  `vault-docs.conf`, mostrano una barra sopra il prompt e aggiungono il comando `/progetto-vault`.

## Regole

1. La chiave API è un segreto. Non leggerla, non stamparla, non copiarla, non metterla in un
   comando, in un file della mod o in una risposta. Puoi solo controllare che il file esista e
   non sia vuoto (`test -s <file>`).
2. Scrivi ogni file della sezione «File della mod» esattamente come è scritto, byte per byte. Non
   riformattare, non tradurre, non «migliorare» il codice. Il contenuto è tra le righe con quattro
   backtick: la riga di apertura e quella di chiusura non fanno parte del file.
3. Non toccare file fuori da quelli elencati qui, tranne le impostazioni indicate al passo 5.
4. Se un passo non riesce, fermati, di' alla persona cosa è successo e cosa fare. Non inventare
   soluzioni che cambiano il sistema (installare programmi, cambiare permessi) senza il suo sì.

## Passo 1. Controlla i prerequisiti

Esegui e riferisci il risultato di ciascuno:

```bash
claude --version
node --version
git --version
curl --version
```

- `node` deve esistere ed essere nel PATH: il server MCP della mod parte con `node`.
- `git` serve per trovare il progetto e per l'allineamento a ogni commit.
- `curl` serve per l'accesso SSH e per l'HTTPS con un file CA.

Se manca qualcosa, di' alla persona cosa installare e fermati.

## Passo 2. Chiedi alla persona

Fai queste domande, una alla volta, e aspetta la risposta:

1. **Indirizzo del server Company Brain** (`url`), per esempio `https://mio-server:3700`.
2. **Come raggiungerlo**:
   - HTTPS diretto con la CA di sistema;
   - HTTPS diretto con il file della CA del server (`caFile`, un file PEM, per esempio
     `data/tls/ca.pem` del repository di Company Brain);
   - SSH (`sshHost`, nella forma `utente@host`): le richieste passano da `ssh` e `curl` sul server.
     Serve se la rete intercetta l'HTTPS (per esempio un antivirus).
3. **Percorso del file della chiave API** (`apiKeyFile`). Predefinito:
   `~/.claude/company-brain-progetti.key`.

Se la persona non ha ancora la chiave, spiegale che va creata sul server Company Brain, legata alla
persona, con gli ambiti `read` e `projects`. Dalla cartella del server:

```bash
npm run -s cb -- api create claude-code-nome.cognome --scopes read,projects --user nome.cognome > chiave.txt
```

Poi la persona copia `chiave.txt` nel percorso scelto. Non farlo tu e non aprire il file.

Controlla senza leggere il contenuto (sostituisci il percorso, con `~` espanso):

```bash
test -s ~/.claude/company-brain-progetti.key && echo "chiave presente" || echo "chiave mancante o vuota"
```

Con SSH, controlla anche l'accesso senza password (deve stampare `ok`):

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 utente@host echo ok
```

Se chiede una password o fallisce, la persona deve configurare una chiave SSH verso il server
(`ssh-copy-id utente@host`). Con un file CA, controlla che esista (`test -s <caFile>`).

## Passo 3. Scrivi i file

Crea la cartella dei mod locali e la mod. `~` è la cartella dell'utente (su Windows
`C:\Users\<nome>`).

```
~/.claude/mods/
├── .claude-plugin/marketplace.json
└── company-brain-progetti/
    ├── .claude-plugin/plugin.json
    ├── .mcp.json
    ├── README.md
    ├── tsconfig.json
    ├── hooks/hooks.json
    ├── hooks/register.tsx
    ├── hooks/register.test.ts
    ├── server/main.mjs
    ├── server/tools.mjs
    ├── server/tools.test.mjs
    └── types/index.d.ts
```

Se `~/.claude/mods/company-brain-progetti/` esiste già, chiedi alla persona se sovrascriverla.

Se `~/.claude/mods/.claude-plugin/marketplace.json` esiste già, non sovrascriverlo: aggiungi solo
la voce di `company-brain-progetti` al suo array `plugins` e usa il suo `name` al posto di
`mods-locali` nei passi successivi. Altrimenti scrivilo così:

````json
{
  "name": "mods-locali",
  "owner": { "name": "utente" },
  "plugins": [
    {
      "name": "company-brain-progetti",
      "source": "./company-brain-progetti",
      "description": "Un vault di Company Brain per ogni progetto: strumenti MCP, file di vault-docs.conf allineati a ogni commit, barra con lo stato del vault."
    }
  ]
}
````

Poi scrivi i file della sezione «File della mod» in `~/.claude/mods/company-brain-progetti/`.

Controllo: dalla cartella della mod esegui

```bash
claude plugin validate .
node --test server/tools.test.mjs
```

La validazione non deve riportare errori e i test devono passare. Se falliscono, confronta il file
indicato con il testo di questo documento e correggi solo le differenze di copia.

## Passo 4. Installa

```bash
claude plugin marketplace add ~/.claude/mods
claude plugin install company-brain-progetti@mods-locali
```

Controllo: `claude plugin list` mostra `company-brain-progetti`. Se il marketplace esisteva già,
esegui `claude plugin marketplace update mods-locali` prima dell'installazione.

## Passo 5. Configura

Le impostazioni stanno in `~/.claude/settings.json`, sotto `pluginConfigs`. Leggi il file, poi
aggiungi o aggiorna solo questa voce, lasciando intatto il resto:

```json
"pluginConfigs": {
  "company-brain-progetti@mods-locali": {
    "options": {
      "url": "<url del passo 2>",
      "sshHost": "<utente@host, oppure stringa vuota>",
      "apiKeyFile": "<percorso del file della chiave>",
      "caFile": "<percorso del file CA, oppure stringa vuota>"
    }
  }
}
```

- `sshHost` pieno: le richieste passano da SSH, `caFile` non conta.
- `sshHost` vuoto e `caFile` pieno: HTTPS diretto con quella CA.
- Tutti e due vuoti: HTTPS diretto con la CA di sistema.

Controllo: il file è JSON valido (`node -e "JSON.parse(require('fs').readFileSync(process.argv[1],'utf8'))" ~/.claude/settings.json`).

## Passo 6. Verifica

Di' alla persona di chiudere Claude Code e aprire una nuova sessione nella cartella di un progetto
git. Al primo avvio la mod:

1. crea il vault del progetto sul server, se non c'è;
2. crea `vault-docs.conf` nella radice del progetto, con `README.md` e `docs/**/*.md`;
3. scrive `~/.claude/company-brain-progetti.json` (indirizzo, SSH, percorso della chiave e della CA:
   mai la chiave);
4. mostra la barra «◆ Company Brain» sopra il prompt.

Controlli:

- la barra è visibile con il pallino verde (giallo: dati vecchi; rosso: server non raggiungibile);
- `/progetto-vault` nasconde e mostra la barra;
- `vault-docs.conf` esiste nella radice del progetto;
- dopo un commit, alla fine del turno la barra mostra `✔ ora +N −M`.

## Problemi comuni

| Sintomo | Causa probabile | Cosa fare |
| --- | --- | --- |
| Barra rossa «vault non disponibile» | server spento o indirizzo sbagliato | controlla `url`, prova `curl -k <url>` |
| «chiave API non valida o scaduta» | chiave sbagliata o revocata | crea una nuova chiave (passo 2) |
| «accesso negato dal server» | mancano gli ambiti | la chiave deve avere `read,projects` |
| «SSH verso … non riuscito» | accesso SSH con password o host errato | passo 2, controllo SSH |
| «il file della chiave … è vuoto» | file vuoto o percorso errato | controlla `apiKeyFile` |
| «versione precedente» | server Company Brain vecchio | aggiorna il server |
| Nessuno strumento `brain_*` | `node` non nel PATH o chiave senza `read` | passo 1 e passo 2 |
| Nessuna barra | sessione fuori da un progetto, o plugin non attivo | apri un progetto, `claude plugin list` |

## Disinstallazione

```bash
claude plugin uninstall company-brain-progetti@mods-locali
```

Poi, se la persona vuole, cancella `~/.claude/mods/company-brain-progetti/`, la voce in
`pluginConfigs` e `~/.claude/company-brain-progetti.json`. La chiave e il file `vault-docs.conf`
nei progetti restano: cancellali solo se la persona lo chiede.

## Passo 7. Riferisci

Di' alla persona in poche righe: prerequisiti trovati, file scritti, esito di validazione e test,
installazione, impostazioni scelte (senza la chiave) e cosa fare per la verifica del passo 6.

## File della mod

### `.claude-plugin/plugin.json`

````json
{
  "name": "company-brain-progetti",
  "version": "0.7.0",
  "description": "Un vault di Company Brain per ogni progetto: gli strumenti MCP di Company Brain sul vault e sugli altri progetti, i file di vault-docs.conf allineati nel vault a ogni commit, una barra con lo stato del vault.",
  "userConfig": {
    "url": {
      "type": "string",
      "title": "Indirizzo di Company Brain",
      "description": "Indirizzo del server, per esempio https://tuo-server:3700",
      "default": "https://tuo-server:3700"
    },
    "sshHost": {
      "type": "string",
      "title": "Accesso via SSH",
      "description": "utente@host del server: le richieste passano da SSH e curl sul server (per una rete dove l'HTTPS è intercettato). Vuoto: HTTPS diretto (con il file della CA, se indicato).",
      "default": ""
    },
    "apiKeyFile": {
      "type": "string",
      "title": "File della chiave API",
      "description": "File con la chiave di un account API con gli ambiti «read» e «projects», legato alla persona (una riga). ~ è la cartella dell'utente.",
      "default": "~/.claude/company-brain-progetti.key"
    },
    "caFile": {
      "type": "string",
      "title": "File della CA del server",
      "description": "Certificato della CA che firma il certificato del server (PEM), per l'HTTPS diretto senza SSH. Vuoto: CA di sistema.",
      "default": ""
    }
  },
  "types": "./types/index.d.ts"
}
````

### `.mcp.json`

````json
{
  "mcpServers": {
    "docs": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server/main.mjs"]
    }
  }
}
````

### `hooks/hooks.json`

````json
{ "modules": ["./register.tsx"] }
````

### `tsconfig.json`

````json
{
  "extends": "./.claude-plugin/types/tsconfig.json"
}
````

### `types/index.d.ts`

````ts
/** The server's state of a project vault: GET /api/projects/:slug/status. */
export type VaultStatus = {
  vault: { id: string; slug: string; name: string }
  pages: { pages: number; inbox: number; sections: number; tokens: number; updatedAt: string | null }
  /** Top-level documents by ingestion state (stored, parsed, indexed, embedded, compiled, failed). */
  sources: Record<string, number>
  sourcesLast24h: number
  work: { queued: number; running: number; failed: number }
  dream: { night: string; status: 'running' | 'paused' | 'done' | 'failed'; step: string | null; startedAt: string; finishedAt: string | null; tokens: number } | null
}

/** One read from or write into the vault by Claude in this session. */
export type Flow = { at: number; dir: 'read' | 'write'; bytes: number }

/** The last alignment of the files of vault-docs.conf with the vault. */
export type Sync = {
  state: 'running' | 'done' | 'failed'
  at: number
  uploaded: number
  removed: number
  /** Files over the size limit or refused by the server. */
  skipped: number
  error: string | null
}

/** What the band draws. */
export type Panel = {
  name: string
  slug: string
  /** Repository files in the vault (from vault-docs.conf). */
  files: number
  git: boolean
  sync: Sync | null
  problem: string | null
  status: VaultStatus | null
  statusAt: number | null
  statusError: string | null
  flows: Flow[]
}

declare module 'claude-code' {
  interface PluginState {
    'company-brain-progetti': { panel: Shaped<Panel | null>; isHidden: boolean }
  }
}
````

### `server/main.mjs`

````js
// Starts the stdio tool server: what .mcp.json runs.
import { serve } from './tools.mjs'

serve()
````

### `server/tools.mjs`

````js
// The MCP server of company-brain-progetti, on stdio: a proxy to Company Brain's own MCP server
// (POST /mcp).
//
// - Every request goes to Company Brain with the key of the mod: the brain_* tools (search,
//   read, capture into the inbox, …), the resources (brain://<vault>/llms.txt) and the prompts.
//   The key decides what Claude sees: the vaults of the user it is bound to, never above
//   "interno".
//
// The hooks write the settings to ~/.claude/company-brain-progetti.json at the start of the
// session; this server reads them at each request. With `sshHost` a request runs as curl on the
// server over SSH (a network that intercepts HTTPS is no obstacle), else as HTTPS from here with
// the CA file of the server. The key travels on stdin or in a header, never on a command line.
// Plain JavaScript with no dependencies.
import { spawn } from 'node:child_process'
import { readFileSync } from 'node:fs'
import https from 'node:https'
import http from 'node:http'
import { homedir } from 'node:os'
import { join } from 'node:path'
import { createInterface } from 'node:readline'
import { fileURLToPath } from 'node:url'

/** The settings the hooks write at the start of each session. */
export const CONFIG_FILE = join(homedir(), '.claude', 'company-brain-progetti.json')

const expand = (p) => String(p ?? '').replace(/^~(?=[\\/])/, homedir())

export function readConfig(file = CONFIG_FILE) {
  try {
    const c = JSON.parse(readFileSync(file, 'utf8'))
    return c && typeof c.url === 'string' && c.url ? c : null
  } catch {
    return null
  }
}

/** One JSON-RPC request to Company Brain's /mcp; resolves `{ status, text }`. */
export async function post(cfg, body, protocol) {
  const key = readFileSync(expand(cfg.keyFile), 'utf8').trim()
  if (!key) throw new Error(`il file della chiave ${cfg.keyFile} è vuoto`)
  const json = JSON.stringify(body)
  const headers = {
    Authorization: `Bearer ${key}`,
    'Content-Type': 'application/json',
    Accept: 'application/json, text/event-stream',
    ...(protocol ? { 'Mcp-Protocol-Version': protocol } : {}),
  }
  const url = new URL('/mcp', cfg.url)
  if (cfg.sshHost) return viaSsh(cfg, url, headers, json)
  return new Promise((resolve, reject) => {
    const lib = url.protocol === 'https:' ? https : http
    const req = lib.request(
      url,
      { method: 'POST', headers: { ...headers, 'Content-Length': Buffer.byteLength(json) }, timeout: 120_000, ...(cfg.caFile ? { ca: readFileSync(expand(cfg.caFile)) } : {}) },
      (res) => {
        const chunks = []
        res.on('data', (c) => chunks.push(c))
        res.on('end', () => resolve({ status: res.statusCode ?? 0, text: Buffer.concat(chunks).toString('utf8') }))
      },
    )
    req.on('timeout', () => req.destroy(new Error('nessuna risposta entro 120 secondi')))
    req.on('error', reject)
    req.end(json)
  })
}

/** curl on the server against its own address; its config, key and body included, on stdin. */
function viaSsh(cfg, url, headers, json) {
  const quote = (s) => `"${s.replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"`
  const port = url.port || (url.protocol === 'https:' ? '443' : '80')
  const config = [
    `url = ${quote(`${url.protocol}//127.0.0.1:${port}/mcp`)}`,
    'request = "POST"',
    ...Object.entries(headers).map(([k, v]) => `header = ${quote(`${k}: ${v}`)}`),
    `data-binary = ${quote(json)}`,
  ].join('\n')
  return new Promise((resolve, reject) => {
    const p = spawn('ssh', ['-o', 'BatchMode=yes', '-o', 'ConnectTimeout=10', cfg.sshHost, "curl -sk --max-time 120 --config - -w '\\n%{http_code}'"], { stdio: ['pipe', 'pipe', 'pipe'], windowsHide: true })
    const out = []
    const err = []
    p.stdout.on('data', (c) => out.push(c))
    p.stderr.on('data', (c) => err.push(c))
    p.on('error', reject)
    p.on('close', (code) => {
      const s = Buffer.concat(out).toString('utf8')
      if (code !== 0 && !s) return reject(new Error(`SSH verso ${cfg.sshHost} non riuscito: ${Buffer.concat(err).toString('utf8').trim().slice(0, 200)}`))
      const cut = s.lastIndexOf('\n')
      resolve({ status: Number(s.slice(cut + 1)) || 0, text: s.slice(0, cut) })
    })
    p.stdin.end(`${config}\n`)
  })
}

/** The JSON-RPC message of a /mcp answer: plain JSON, or the first `data:` event of a stream. */
export function parseAnswer(text) {
  const t = text.trim()
  if (!t) return null
  try {
    return JSON.parse(t)
  } catch {
    const data = t.split(/\r?\n/).filter((l) => l.startsWith('data:')).map((l) => l.slice(5).trim())
    for (const d of data) {
      try {
        return JSON.parse(d)
      } catch {}
    }
    return null
  }
}

const errorText = (status, msg) => msg?.error?.message ?? (status === 401 ? 'chiave API non valida o scaduta' : status === 403 ? 'accesso negato dal server (serve l’ambito di lettura)' : `risposta ${status}`)

/**
 * Makes the proxy: `respond(message)` resolves the reply to one JSON-RPC message (undefined for
 * a notification). `deps` lets the tests stand in for the network and the settings.
 */
export function createProxy(deps = {}) {
  const send = deps.post ?? post
  const config = deps.config ?? readConfig
  const version = deps.version ?? '0'
  let protocol

  /** The remote answer to `message`, or throws with a sentence for the person. */
  async function remote(message) {
    const cfg = config()
    if (!cfg) throw new Error('impostazioni di Company Brain non ancora scritte dalla mod (apri una sessione in un progetto)')
    const r = await send(cfg, message, protocol)
    const msg = parseAnswer(r.text)
    if (r.status >= 400 || !msg) throw new Error(errorText(r.status, msg))
    return msg
  }

  return async function respond(message) {
    if (message?.id === undefined || message.id === null) return undefined
    const { id, method, params } = message
    const result = (value) => ({ jsonrpc: '2.0', id, result: value })
    const failure = (code, text) => ({ jsonrpc: '2.0', id, error: { code, message: text } })
    try {
      switch (method) {
        case 'ping':
          return result({})
        case 'initialize': {
          protocol = typeof params?.protocolVersion === 'string' ? params.protocolVersion : '2025-06-18'
          const base = { protocolVersion: protocol, capabilities: { tools: {}, resources: {}, prompts: {} }, serverInfo: { name: 'company-brain-progetti', version } }
          try {
            const r = await remote(message)
            if (r.result?.protocolVersion) protocol = r.result.protocolVersion
            return result({ ...base, protocolVersion: protocol, ...(r.result?.instructions ? { instructions: r.result.instructions } : {}) })
          } catch (e) {
            return result({ ...base, instructions: `Company Brain non risponde (${e.message}): i suoi strumenti non sono disponibili in questa sessione.` })
          }
        }
        case 'tools/list': {
          let tools = []
          try {
            tools = (await remote(message)).result?.tools ?? []
          } catch {
            // no tools while the server's MCP does not answer (an older key)
          }
          return result({ tools })
        }
        case 'tools/call':
          try {
            const r = await remote(message)
            return r.error ? failure(r.error.code ?? -32603, r.error.message) : result(r.result)
          } catch (e) {
            return result({ content: [{ type: 'text', text: `Company Brain non ha risposto: ${e.message}.` }], isError: true })
          }
        default: {
          const r = await remote(message)
          return r.error ? failure(r.error.code ?? -32603, r.error.message) : result(r.result ?? {})
        }
      }
    } catch (e) {
      return failure(-32603, `Company Brain non ha risposto: ${e.message}.`)
    }
  }
}

const versionOf = () => {
  try {
    const { version } = JSON.parse(readFileSync(fileURLToPath(new URL('../.claude-plugin/plugin.json', import.meta.url)), 'utf8'))
    return typeof version === 'string' ? version : '0'
  } catch {
    return '0'
  }
}

/** Serves the proxy over newline-delimited JSON on stdin and stdout until stdin closes. */
export const serve = (input = process.stdin, output = process.stdout) => {
  const respond = createProxy({ version: versionOf() })
  createInterface({ input }).on('line', (line) => {
    if (!line.trim()) return
    let message
    try {
      message = JSON.parse(line)
    } catch {
      output.write(`${JSON.stringify({ jsonrpc: '2.0', id: null, error: { code: -32700, message: 'Parse error' } })}\n`)
      return
    }
    // requests answer as they finish: the client matches them by id
    void respond(message).then((reply) => reply && output.write(`${JSON.stringify(reply)}\n`))
  })
}
````

### `server/tools.test.mjs`

````js
// The MCP proxy against a pretend Company Brain: `node --test server/`.
import assert from 'node:assert/strict'
import { test } from 'node:test'
import { createProxy, parseAnswer } from './tools.mjs'

const CFG = { url: 'https://brain.test:3700', sshHost: '', keyFile: '~/k', caFile: '' }

/** A pretend /mcp: answers by method, records what it got. */
function brain(answers) {
  const seen = []
  const post = async (cfg, body, protocol) => {
    seen.push({ body, protocol })
    const a = answers[body.method]
    if (typeof a === 'function') return a(body)
    if (!a) return { status: 200, text: JSON.stringify({ jsonrpc: '2.0', id: body.id, error: { code: -32601, message: 'no' } }) }
    return { status: 200, text: JSON.stringify({ jsonrpc: '2.0', id: body.id, result: a }) }
  }
  return { post, seen }
}

test('initialize keeps the server instructions and offers tools, resources and prompts', async () => {
  const b = brain({ initialize: { protocolVersion: '2025-06-18', instructions: 'Company Brain è la memoria.', capabilities: { tools: {} } } })
  const respond = createProxy({ post: b.post, config: () => CFG, version: '0.4.0' })
  const r = await respond({ jsonrpc: '2.0', id: 1, method: 'initialize', params: { protocolVersion: '2025-06-18' } })
  assert.equal(r.result.instructions, 'Company Brain è la memoria.')
  assert.deepEqual(r.result.capabilities, { tools: {}, resources: {}, prompts: {} })
  assert.equal(r.result.serverInfo.name, 'company-brain-progetti')
})

test('tools/list: the server tools; the server down leaves none', async () => {
  const b = brain({ 'tools/list': { tools: [{ name: 'brain_search', inputSchema: { type: 'object' } }] } })
  const respond = createProxy({ post: b.post, config: () => CFG })
  const r = await respond({ jsonrpc: '2.0', id: 2, method: 'tools/list' })
  assert.deepEqual(r.result.tools.map((t) => t.name), ['brain_search'])
  const down = createProxy({ post: async () => ({ status: 403, text: '{"error":{"message":"Questo account API non ha l\'ambito di lettura."}}' }), config: () => CFG })
  assert.deepEqual((await down({ jsonrpc: '2.0', id: 3, method: 'tools/list' })).result.tools, [])
})

test('tools/call goes to the server; a failure is a tool error', async () => {
  const b = brain({ 'tools/call': (body) => ({ status: 200, text: `event: message
data: ${JSON.stringify({ jsonrpc: '2.0', id: body.id, result: { content: [{ type: 'text', text: 'trovato' }] } })}

` }) })
  const respond = createProxy({ post: b.post, config: () => CFG })
  const remote = await respond({ jsonrpc: '2.0', id: 5, method: 'tools/call', params: { name: 'brain_search', arguments: { query: 'octree' } } })
  assert.equal(remote.result.content[0].text, 'trovato')
  assert.equal(b.seen[0].body.params.arguments.query, 'octree')
  const broken = createProxy({ post: async () => { throw new Error('SSH verso pi@x non riuscito') }, config: () => CFG })
  const r = await broken({ jsonrpc: '2.0', id: 6, method: 'tools/call', params: { name: 'brain_read', arguments: {} } })
  assert.equal(r.result.isError, true)
  assert.match(r.result.content[0].text, /SSH verso pi@x non riuscito/)
})

test('resources and prompts pass through; the protocol of initialize rides along; notifications get no reply', async () => {
  const b = brain({ initialize: { protocolVersion: '2025-03-26' }, 'resources/read': { contents: [{ uri: 'brain://p/llms.txt', text: 'mappa' }] } })
  const respond = createProxy({ post: b.post, config: () => CFG })
  await respond({ jsonrpc: '2.0', id: 7, method: 'initialize', params: { protocolVersion: '2025-06-18' } })
  const r = await respond({ jsonrpc: '2.0', id: 8, method: 'resources/read', params: { uri: 'brain://p/llms.txt' } })
  assert.equal(r.result.contents[0].text, 'mappa')
  assert.equal(b.seen.at(-1).protocol, '2025-03-26')
  assert.equal(await respond({ jsonrpc: '2.0', method: 'notifications/initialized' }), undefined)
})

test('without the settings of the hooks the proxy says why', async () => {
  const respond = createProxy({ post: async () => assert.fail('no request'), config: () => null })
  const r = await respond({ jsonrpc: '2.0', id: 9, method: 'prompts/list' })
  assert.match(r.error.message, /impostazioni di Company Brain non ancora scritte/)
})

test('parseAnswer reads plain JSON and the first data event of a stream', () => {
  assert.deepEqual(parseAnswer('{"a":1}'), { a: 1 })
  assert.deepEqual(parseAnswer('event: message\ndata: {"b":2}\n\n'), { b: 2 })
  assert.equal(parseAnswer(''), null)
})
````

### `hooks/register.tsx`

````tsx
// Company Brain for Claude Code: one vault per project (a git repository), Company Brain's MCP
// tools on it and on the other vaults the person may read, the documentation files of the
// repository kept aligned in the vault, and a band above the prompt with the state of the vault.
//
// session.start   finds the project (git remote name, else the folder; a folder outside git
//                 counts when it has a project file), creates its vault on the first session,
//                 creates vault-docs.conf when the project has none, reads the files the vault
//                 holds, writes the settings the MCP proxy reads
// prompt.compose  adds the vault, its files and the rules to the system prompt
// prompt.submit   on the person's first prompt, hidden context: search the vault before working;
//                 a server that did not answer at start is asked again
// turn.complete   after a turn that ends on a new commit (HEAD moved during the session), aligns
//                 the vault with the files vault-docs.conf names, in the background: new and
//                 changed files are uploaded, files no longer named or gone are deleted
// tools           the MCP server of .mcp.json (server/tools.mjs) proxies Company Brain's /mcp
//                 (brain_search, brain_read, brain_capture, …)
// tool.call       counts reads and writes of the vault for the band
// /progetto-vault shows or hides the band
//
// vault-docs.conf lives at the root of the project, one rule per line in the syntax of
// .gitignore (files, folders, wildcards, ! to exclude, # comments). git matches it against the
// files it tracks (`git ls-files -c -i --exclude-from`), so the rules mean what they mean there.
//
// Requests go to the server's /api/projects routes with the key of the mod (scopes read and
// projects, bound to the person). With `sshHost` they run as curl on the server over SSH, so a
// network that intercepts HTTPS (an antivirus) is no obstacle; with `caFile` they run as curl
// here, trusting the server's CA. The key travels on stdin, never on a command line.
import { atom, read, update } from 'claude-code'
import type { EngineInterface, Register } from 'claude-code'

import type { Flow, Panel, Sync, VaultStatus } from '../types'

type Engine = EngineInterface

/** A repository file the vault holds: GET /api/projects/:slug/files. */
type VaultFile = { path: string; sourceId: string; status: string; commit?: string }
type Project = { slug: string; name: string; vaultId: string; root: string; git: boolean; files: VaultFile[] }

// The tools as Claude Code names those of a plugin's own MCP server (.mcp.json, server "docs").
const MCP = 'mcp__plugin_company-brain-progetti_docs__'
const BRAIN = (name: string) => `${MCP}brain_${name}`
/** Company Brain's tools the model reaches without ToolSearch: the ones of every session. */
const EAGER = [BRAIN('overview'), BRAIN('search'), BRAIN('read'), BRAIN('capture')]
/** Tools that write into a vault; every other tool of the server reads. */
const WRITES = new Set([BRAIN('capture'), BRAIN('propose_edit')])
const PANEL_COMMAND = 'progetto-vault'
/** Refresh of the server's state while the session has a project. */
const STATUS_EVERY_MS = 60_000
/** The window of the rates in the band. */
const RATE_WINDOW_MS = 5 * 60_000

const CONF = 'vault-docs.conf'
const CONF_DEFAULT = [
  '# vault-docs.conf: i file di documentazione che company-brain-progetti allinea nel vault del',
  '# progetto in Company Brain a ogni commit (file nuovi e cambiati caricati, file tolti cancellati).',
  '# Una regola per riga, come in .gitignore: file singoli, cartelle (docs/), wildcard (*.md,',
  '# docs/**/*.pdf), ! per escludere, # per i commenti. Contano solo i file tracciati da git,',
  '# al massimo 4 MiB ciascuno.',
  'README.md',
  'docs/**/*.md',
  '',
].join('\n')
/** The largest file the mod sends: what one $.fs.read returns. */
const FILE_MAX_BYTES = 4 * 1024 ** 2

// A folder outside git is a project when it holds one of these.
const MARKERS = ['package.json', 'Cargo.toml', 'pyproject.toml', 'requirements.txt', 'setup.py', 'go.mod', 'pom.xml', 'build.gradle', 'build.gradle.kts', 'CMakeLists.txt', 'Makefile', 'composer.json', 'Gemfile', 'deno.json', 'pubspec.yaml', 'mix.exs']

// What the band draws: written by the hooks, read while drawing. Change the tag when its shape
// changes: an earlier version's value outlives a reload.
const panel = atom({ plugin: 'company-brain-progetti', key: 'panel' } as const, null as Panel | null, { shape: 'panel-3' })
/** The band above the prompt, hidden by its button or by /progetto-vault. */
const isHidden = atom({ plugin: 'company-brain-progetti', key: 'isHidden' } as const, false)

// The module's state: options, the project of the session. A reload starts it over and
// session.start fills it again.
const cfg = { url: '', sshHost: '', keyFile: '', caFile: '' }
let project: Project | null = null
let problem: string | null = null
/** The commit HEAD pointed at when the session last looked: a different one is a new commit. */
let seenHead: string | undefined
/** False until the first load: a repository with no commit yet has no `seenHead` either. */
let headKnown = false
/** The person's first prompt of the session gets the context. */
let firstPrompt = true
/**
 * When the last load started. Without a project (the folder became a repository during the
 * session, or the server did not answer) it runs again at most once a minute.
 */
let triedAt = 0
let statusTimer: { cancel?: () => void } | null = null
/** The last alignment, also while it runs. */
let sync: Sync | null = null

async function home($: Engine): Promise<string> {
  return (await $.env.get('USERPROFILE')) ?? (await $.env.get('HOME')) ?? ''
}
const expand = (p: string, h: string) => p.replace(/^~(?=[\\/])/, h)

async function apiKey($: Engine): Promise<string> {
  const key = (await $.fs.read(expand(cfg.keyFile, await home($)))).trim()
  if (!key) throw new Error(`il file della chiave ${cfg.keyFile} è vuoto`)
  return key
}

const quote = (s: string) => `"${s.replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"`

/** One request to the server; JSON in and out. */
async function call($: Engine, method: 'GET' | 'POST' | 'PUT' | 'DELETE', path: string, body?: unknown): Promise<{ status: number; data: any }> {
  const key = await apiKey($)
  const json = body === undefined ? undefined : JSON.stringify(body)
  const lines = (url: string) => [
    `url = ${quote(url)}`,
    `request = ${quote(method)}`,
    `header = ${quote(`Authorization: Bearer ${key}`)}`,
    ...(json ? [`header = "Content-Type: application/json"`, `data-binary = ${quote(json)}`] : []),
  ]
  let r: { exitCode: number; stdout: string; stderr: string }
  if (cfg.sshHost) {
    // curl on the server against its own address, its config (key and body included) on stdin
    const port = /^https?:\/\/[^/:]+:(\d+)/.exec(cfg.url)?.[1] ?? (cfg.url.startsWith('https') ? '443' : '80')
    const scheme = cfg.url.startsWith('https') ? 'https' : 'http'
    r = await $.process.run(
      ['ssh', '-o', 'BatchMode=yes', '-o', 'ConnectTimeout=10', cfg.sshHost, "curl -sk --max-time 60 --config - -w '\\n%{http_code}'"],
      { stdin: lines(`${scheme}://127.0.0.1:${port}${path}`).join('\n') + '\n', timeoutMs: 75_000 },
    )
    if (r.exitCode !== 0 && !r.stdout) throw new Error(`SSH verso ${cfg.sshHost} non riuscito: ${r.stderr.trim().slice(0, 200)}`)
  } else if (cfg.caFile) {
    // curl here, trusting the CA of the server
    const config = [...lines(cfg.url + path), `cacert = ${quote(expand(cfg.caFile, await home($)))}`].join('\n')
    r = await $.process.run(['curl', '-s', '--max-time', '60', '--config', '-', '-w', '\n%{http_code}'], { stdin: config + '\n', timeoutMs: 75_000 })
    if (r.exitCode !== 0 && !r.stdout) throw new Error(`HTTPS verso ${cfg.url} non riuscito: ${r.stderr.trim().slice(0, 200) || `curl ${r.exitCode}`}`)
  } else {
    const f = await $.http.fetch(cfg.url + path, {
      method,
      headers: { authorization: `Bearer ${key}`, ...(json ? { 'content-type': 'application/json' } : {}) },
      ...(json ? { body: json } : {}),
    })
    return { status: f.status, data: parse(f.text) }
  }
  const cut = r.stdout.lastIndexOf('\n')
  return { status: Number(r.stdout.slice(cut + 1)) || 0, data: parse(r.stdout.slice(0, cut)) }
}

function message(r: { status: number; data: any }): string {
  return r.data?.error?.message ?? `risposta ${r.status}`
}

/** The settings of the MCP proxy (server/tools.mjs), which runs as its own process. */
async function writeProxyConfig($: Engine): Promise<void> {
  const h = await home($)
  if (!h) return
  await $.fs.write(`${h}/.claude/company-brain-progetti.json`, `${JSON.stringify({ url: cfg.url, sshHost: cfg.sshHost, keyFile: cfg.keyFile, caFile: cfg.caFile }, null, 2)}\n`)
}

/**
 * The project of the session: the git repository it runs in, else a folder with a project file
 * (not the home folder, not a drive root). Null elsewhere.
 */
async function identify($: Engine): Promise<{ slug: string; name: string; root: string; git: boolean } | null> {
  const root = await $.session.root()
  const top = await $.process.run(['git', 'rev-parse', '--show-toplevel'], { cwd: root })
  let base: string
  let fromRemote: string | undefined
  if (top.exitCode === 0) {
    base = top.stdout.trim()
    const remote = await $.process.run(['git', 'remote', 'get-url', 'origin'], { cwd: root })
    fromRemote = remote.exitCode === 0 ? remote.stdout.trim().split(/[\\/:]/).pop()?.replace(/\.git$/, '') : undefined
  } else {
    const norm = (p: string) => p.replace(/\\/g, '/').replace(/\/+$/, '').toLowerCase()
    if (!root || norm(root) === norm(await home($)) || /^([a-z]:)?$/.test(norm(root))) return null
    let found = false
    for (const m of MARKERS) if (await $.fs.exists(`${root}/${m}`)) found = true
    if (!found) return null
    base = root.replace(/[\\/]+$/, '')
  }
  const name = (fromRemote || base.split(/[\\/]/).pop() || '').trim()
  const slug = name
    .toLowerCase()
    .normalize('NFD')
    .replace(/[̀-ͯ]/g, '')
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '')
    .slice(0, 63)
  return slug.length >= 2 ? { slug, name, root: base, git: top.exitCode === 0 } : null
}

/** The commit HEAD points at, or undefined in a repository with no commit yet. */
async function head($: Engine, root: string): Promise<string | undefined> {
  const r = await $.process.run(['git', 'rev-parse', 'HEAD'], { cwd: root })
  const sha = r.stdout.trim()
  return r.exitCode === 0 && /^[0-9a-f]{7,40}$/.test(sha) ? sha : undefined
}

/** vault-docs.conf at the root of the project; written with the default rules when missing. */
async function ensureConf($: Engine, root: string): Promise<string> {
  const path = `${root}/${CONF}`
  if (!(await $.fs.exists(path))) await $.fs.write(path, CONF_DEFAULT)
  return path
}

/** The repository files of the vault, or why the server did not list them. */
async function listFiles($: Engine, slug: string): Promise<VaultFile[]> {
  const r = await call($, 'GET', `/api/projects/${slug}/files`)
  if (r.status === 404 && !r.data?.error?.message?.includes('Progetto')) throw new Error('il server non ha ancora i file di progetto (versione precedente)')
  if (r.status !== 200) throw new Error(message(r))
  return r.data.files as VaultFile[]
}

async function load($: Engine): Promise<void> {
  triedAt = await $.clock.now()
  const id = await identify($)
  if (!id) {
    project = null
    problem = null
    return
  }
  try {
    let r = await call($, 'GET', `/api/projects/${id.slug}`)
    if (r.status === 404) {
      const c = await call($, 'POST', '/api/projects', { slug: id.slug, name: id.name })
      if (c.status !== 201 && c.status !== 200) throw new Error(message(c))
      r = await call($, 'GET', `/api/projects/${id.slug}`)
    }
    if (r.status !== 200) throw new Error(message(r))
    await ensureConf($, id.root)
    // a server without the file routes keeps the vault and the band; the alignment says why not
    const files = await listFiles($, id.slug).catch(async (err: unknown) => {
      sync = { state: 'failed', at: await $.clock.now(), uploaded: 0, removed: 0, skipped: 0, error: String((err as Error)?.message ?? err).slice(0, 200) }
      return [] as VaultFile[]
    })
    project = { slug: id.slug, name: r.data.vault.name, vaultId: r.data.vault.id, root: id.root, git: id.git, files }
    problem = null
    if (!headKnown && id.git) {
      seenHead = await head($, id.root)
      headKnown = true
    }
  } catch (err) {
    project = null
    problem = `${id.name}: ${(err as Error).message ?? err}`
  }
}

/** No project yet: look again, at most once a minute. */
async function retry($: Engine): Promise<void> {
  if (project || (await $.clock.now()) - triedAt < 60_000) return
  await load($)
  showStatus($)
}

const lines = (s: string) => s.split(/\r?\n|\0/).map((l) => l.trim()).filter(Boolean)

/**
 * Aligns the vault with the files vault-docs.conf names at `commit`: uploads the files the vault
 * lacks and those changed since the commit the vault last saw them at, deletes the files no
 * longer named or no longer in the repository. The server ignores bytes it already has.
 */
async function align($: Engine, p: Project, commit: string): Promise<Omit<Sync, 'state' | 'at'>> {
  const conf = await ensureConf($, p.root)
  const ls = await $.process.run(['git', 'ls-files', '-z', '--cached', '--ignored', `--exclude-from=${conf}`], { cwd: p.root })
  if (ls.exitCode !== 0) throw new Error(`git ls-files: ${ls.stderr.trim().slice(0, 120)}`)
  const named = new Set(lines(ls.stdout).filter((f) => f !== CONF))
  const held = await listFiles($, p.slug)
  const byPath = new Map<string, VaultFile[]>()
  for (const f of held) byPath.set(f.path, [...(byPath.get(f.path) ?? []), f])

  // what changed since each commit the vault knows; an unknown commit: every file of it again
  const changedSince = new Map<string, Set<string> | null>()
  for (const c of new Set(held.map((f) => f.commit).filter((c): c is string => !!c))) {
    const d = await $.process.run(['git', 'diff', '--name-only', c, commit], { cwd: p.root })
    changedSince.set(c, d.exitCode === 0 ? new Set(lines(d.stdout)) : null)
  }
  const owed = [...named].filter((path) => {
    const copies = byPath.get(path)
    // missing, or two sources for one path (an old one the server could not delete yet)
    if (!copies || copies.length > 1) return true
    const c = copies[0]!.commit
    if (!c || c === commit) return !c
    const changed = changedSince.get(c)
    return changed === null || !!changed?.has(path)
  })

  let uploaded = 0
  let removed = 0
  let skipped = 0
  for (const path of owed) {
    const full = `${p.root}/${path}`
    const stat = await $.fs.stat(full).catch(() => null)
    if (!stat || stat.kind !== 'file') continue
    if (stat.size === 0 || stat.size > FILE_MAX_BYTES) {
      skipped++
      continue
    }
    const { base64 } = await $.fs.read(full, { as: 'bytes' })
    const r = await call($, 'PUT', `/api/projects/${p.slug}/files`, { path, content: base64, commit })
    if (r.status !== 200) {
      // a path the server refuses (characters, size): the others go on
      if (r.status === 400 || r.status === 413) {
        skipped++
        continue
      }
      throw new Error(`${path}: ${message(r)}`)
    }
    if (r.data.created) uploaded++
    removed += r.data.removed ?? 0
  }
  for (const path of byPath.keys()) {
    if (named.has(path)) continue
    const r = await call($, 'DELETE', `/api/projects/${p.slug}/files?path=${encodeURIComponent(path)}`)
    if (r.status !== 200) throw new Error(`${path}: ${message(r)}`)
    removed += r.data.removed ?? 0
  }
  return { uploaded, removed, skipped, error: null }
}

/** Runs one alignment in the background; a failure lets the next turn's end try again. */
function startAlign($: Engine, commit: string, before: string | undefined): void {
  const p = project
  if (!p) return
  $.clock.after(0, async () => {
    const at = await $.clock.now()
    sync = { state: 'running', at, uploaded: 0, removed: 0, skipped: 0, error: null }
    showStatus($)
    try {
      const r = await align($, p, commit)
      sync = { state: 'done', at: await $.clock.now(), ...r }
      if (project?.slug === p.slug) project = { ...project, files: await listFiles($, p.slug) }
      if (r.uploaded || r.removed) {
        $.ui.toast(`Company Brain: vault «${p.slug}» allineato (${r.uploaded} caricati, ${r.removed} tolti${r.skipped ? `, ${r.skipped} saltati` : ''}).`)
        $.clock.after(5_000, () => void refreshStatus($).catch(() => {}))
      }
    } catch (err) {
      seenHead = before
      sync = { state: 'failed', at: await $.clock.now(), uploaded: 0, removed: 0, skipped: 0, error: String((err as Error)?.message ?? err).slice(0, 200) }
    }
    showStatus($)
  })
}

function showStatus($: Engine) {
  void syncPanel($).catch(() => {})
  if (problem) return $.ui.status(`Company Brain: vault non disponibile (${problem.slice(0, 80)})`)
  if (!project) return $.ui.status(undefined)
  const state = sync?.state === 'running' ? ' · allineamento in corso' : sync?.state === 'failed' ? ` · allineamento non riuscito (${sync.error?.slice(0, 60)})` : ''
  $.ui.status(`Company Brain · ${project.name}: ${project.files.length} file da ${CONF}${state}`)
}

// ---------------------------------------------------------------------------
// The band: the project, the server's state of its vault, reads and writes of this session
// ---------------------------------------------------------------------------

/** The band's copy of the project; the flows and the server's state stay as they are. */
async function syncPanel($: Engine): Promise<void> {
  await update($, panel, (p) =>
    project
      ? {
          name: project.name,
          slug: project.slug,
          files: new Set(project.files.map((f) => f.path)).size,
          git: project.git,
          sync,
          problem: null,
          status: p?.slug === project.slug ? p.status : null,
          statusAt: p?.slug === project.slug ? p.statusAt : null,
          statusError: p?.slug === project.slug ? p.statusError : null,
          flows: p?.slug === project.slug ? p.flows : [],
        }
      : problem
        ? { name: '', slug: '', files: 0, git: false, sync: null, problem, status: null, statusAt: null, statusError: null, flows: [] }
        : null,
  )
}

/** The server's state of the vault (GET /api/projects/:slug/status). */
async function refreshStatus($: Engine): Promise<void> {
  if (!project) return
  const slug = project.slug
  let status: VaultStatus | null = null
  let error: string | null = null
  try {
    const r = await call($, 'GET', `/api/projects/${slug}/status`)
    if (r.status === 200) status = r.data as VaultStatus
    else error = r.status === 404 ? 'il server non ha ancora lo stato del vault (versione precedente)' : message(r)
  } catch (err) {
    error = String((err as Error)?.message ?? err).slice(0, 120)
  }
  const at = await $.clock.now()
  await update($, panel, (p) => (p && p.slug === slug ? { ...p, status: status ?? p.status, statusAt: status ? at : p.statusAt, statusError: error } : p))
}

/** A read or a write of the vault, for the rates of the band. */
async function record($: Engine, dir: Flow['dir'], bytes: number): Promise<void> {
  const at = await $.clock.now()
  await update($, panel, (p) => (p ? { ...p, flows: [...p.flows.filter((f) => at - f.at < 60 * 60_000), { at, dir, bytes }].slice(-500) } : p))
}

const kb = (n: number) => (n < 1024 ? `${n} B` : `${(n / 1024).toFixed(n < 10_240 ? 1 : 0)} kB`)
const perMin = (n: number) => (n / (RATE_WINDOW_MS / 60_000)).toFixed(1).replace('.', ',')
const ago = (ms: number) => (ms < 90_000 ? 'ora' : ms < 3_600_000 ? `${Math.round(ms / 60_000)} min fa` : `${Math.round(ms / 3_600_000)} h fa`)

export const register: Register = (on, options) => {
  cfg.url = String(options.url ?? '').replace(/\/+$/, '')
  cfg.sshHost = String(options.sshHost ?? '').trim()
  cfg.keyFile = String(options.apiKeyFile ?? '')
  cfg.caFile = String(options.caFile ?? '').trim()

  // It counts the reads and writes of the vault for the band.
  on('tool.call', async ($, e, next) => {
    const ran = await next(e)
    const tool = String(e.tool)
    const resource = tool === 'ReadMcpResourceTool' && String((e as { server?: unknown }).server ?? '').includes('company-brain-progetti')
    if (project && (tool.startsWith(MCP) || resource) && ran.deny === undefined && !ran.isError) {
      const journalWrite = tool === BRAIN('journal') && (e as { action?: unknown }).action !== 'leggi'
      const write = WRITES.has(tool) || journalWrite
      const { tool: _t, tool_use_id: _i, ...args } = e as Record<string, unknown>
      await record($, write ? 'write' : 'read', write ? JSON.stringify(args).length : JSON.stringify(ran.result ?? '').length)
      if (write) $.clock.after(5_000, () => void refreshStatus($).catch(() => {}))
    }
    return ran
  }).catch(($, e, next) => next(e))

  on('session.start', async ($, e, next) => {
    await $.command.register({ name: PANEL_COMMAND, description: 'Mostra o nasconde lo stato del vault del progetto in Company Brain' })
    await writeProxyConfig($).catch(() => {})
    await load($)
    showStatus($)
    if (project) {
      void refreshStatus($).catch(() => {})
      statusTimer?.cancel?.()
      statusTimer = $.clock.every(STATUS_EVERY_MS, () => void refreshStatus($).catch(() => {})) as never
    }
    return next(e)
  })

  on('prompt.compose', async ($, e, next) => {
    const r = await next(e)
    if (!project && !problem) return r
    const paths = [...new Set(project?.files.map((f) => f.path) ?? [])]
    const text = project
      ? [
          '# Documentazione di progetto in Company Brain',
          `Questo è il progetto «${project.name}». La sua memoria vive nel vault \`${project.slug}\` di Company Brain.`,
          project.git
            ? `I file di documentazione del repository elencati in \`${CONF}\` (radice del progetto, una regola per riga come in .gitignore) sono documenti del vault: la mod li allinea da sola a ogni commit, in background (file nuovi e cambiati caricati, file tolti cancellati). Per aggiungere o togliere documenti cambia \`${CONF}\`: vale dal commit successivo. Non caricarli a mano.`
            : `\`${CONF}\` elenca i file di documentazione del progetto, ma fuori da git la mod non li allinea.`,
          paths.length
            ? `Nel vault ora: ${paths.length} file (${paths.slice(0, 20).map((f) => `\`${f}\``).join(', ')}${paths.length > 20 ? ', …' : ''}).`
            : 'Nel vault non c\'è ancora nessun file del repository.',
          '',
          '## Memoria di Company Brain (strumenti brain_*)',
          `- Tutto il vault del progetto: la risorsa \`brain://${project.slug}/llms.txt\` (mappa delle pagine con il costo in token), poi ${BRAIN('read')}. Per cercare: ${BRAIN('search')} con vaults ["${project.slug}"].`,
          `- Gli altri progetti: ${BRAIN('search')} senza vaults (o con gli slug di altri vault di progetto) trova analisi, decisioni e concetti già scritti. Prima di progettare una parte nuova cerca se esiste già; valuta se si applica a questo progetto e dillo alla persona, citando la pagina con ^[id].`,
          `- Note per la memoria: ${BRAIN('capture')} con vault "${project.slug}" salva decisioni, scoperte e sintesi utili nella inbox del vault; Company Brain le compila in pagine (ingestion) e il ciclo notturno le collega. Scrivi note brevi e fattuali, con le fonti; non copiare il codice.`,
          '- Vedi solo pagine «pubblico» e «interno» e solo i vault che la persona può leggere.',
          'Non mettere mai nei documenti credenziali, chiavi, token o dati personali.',
        ].join('\n')
      : `# Documentazione di progetto in Company Brain\nNon disponibile in questa sessione: ${problem}. Non provare a leggerla o scriverla; se la persona la chiede, riferisci il problema.`
    return { ...r, sections: [...r.sections, { id: `${$.plugin.name}:docs`, text, scope: 'session' as const }] }
  })

  // The model reaches for them on any turn: their schemas sit in the prompt, not behind ToolSearch
  // where the engine puts an MCP server's tools.
  for (const tool of EAGER) {
    on('tool.describe', { tool }, async ($, e, next) => ({ ...(await next(e)), isDeferred: false }))
  }

  // The person's prompt: a server that failed at start is asked again; the first prompt of the
  // session carries, hidden, the order to search the vault before working.
  on('prompt.submit', async ($, e, next) => {
    const o = e.origin as { kind?: string } | undefined
    if (e.command || e.isMeta || e.turnId) return next(e)
    if (!(o?.kind === 'composer' || o?.kind === 'bridge' || o?.kind === 'sdk')) return next(e)
    await retry($)
    if (!project || !firstPrompt) return next(e)
    firstPrompt = false
    const text = `Documentazione di progetto in Company Brain: prima di rispondere o di toccare il codice, cerca con ${BRAIN('search')} (vaults ["${project.slug}"]) ciò che il vault sa della richiesta. Se la richiesta riguarda una parte che altri progetti potrebbero avere già risolto, cercala anche senza vaults.`
    return next({ ...e, context: [...(e.context ?? []), text] })
  }).catch(($, e, next) => next(e))

  // The end of a turn of the main loop: a new commit aligns the vault with vault-docs.conf.
  on('turn.complete', async ($, e, next) => {
    const r = await next(e)
    if (e.agentId !== undefined || e.reason !== 'answer') return r
    await retry($)
    if (!project?.git || sync?.state === 'running') return r
    const now = await head($, project.root)
    if (!now || now === seenHead) return r
    const before = seenHead
    seenHead = now
    startAlign($, now, before)
    return r
  }).catch(($, e, next) => next(e))

  // /progetto-vault: the band above the prompt shows or hides.
  on('command.run', { command: PANEL_COMMAND }, async ($) => {
    if (!project) return { text: `Nessun vault di progetto in questa sessione${problem ? `: ${problem}` : ''}.` }
    const hidden = await update($, isHidden, (v) => !v)
    void refreshStatus($).catch(() => {})
    return { text: hidden ? 'Monitor di Company Brain nascosto. /progetto-vault lo mostra di nuovo.' : `Monitor del vault «${project.slug}» sopra il prompt.` }
  })

  // The band above the prompt. It stacks over what the plugins beneath draw (context-bar), so
  // both show whatever the order of the plugins.
  on('ui.render', { component: 'AbovePrompt' }, async ($, e, next) => {
    const p = await read($, panel)
    if (!p || e.props.hasSurvey || (await read($, isHidden))) return next(e)
    const rest = await next(e)
    const ui = $.ui.resolve(e)
    return (
      <ui.Box flexDirection="column">
        {monitor(ui, p, await $.clock.now(), { hide: () => void update($, isHidden, () => true), refresh: () => void refreshStatus($).catch(() => {}) })}
        {rest}
      </ui.Box>
    )
  })
}

// ---------------------------------------------------------------------------
// The monitor, in the look of context-bar (round frame, bold labels, coloured dots and bars,
// tabular numbers)
// ---------------------------------------------------------------------------

type Ui = { Box: any; Text: any; Button: any }
const SPARK = '▁▂▃▄▅▆▇█'
const SPARK_BUCKETS = 12

/** One glyph per minute of the last 12, oldest first: how much moved in that minute (0 = none). */
function sparkline(flows: Flow[], dir: Flow['dir'], now: number): { glyph: string; n: number }[] {
  const counts = new Array<number>(SPARK_BUCKETS).fill(0)
  for (const f of flows) {
    const age = Math.floor((now - f.at) / 60_000)
    if (f.dir === dir && age >= 0 && age < SPARK_BUCKETS) counts[SPARK_BUCKETS - 1 - age]! += 1
  }
  const max = Math.max(1, ...counts)
  return counts.map((c) => ({ glyph: SPARK[c === 0 ? 0 : Math.max(1, Math.round((c / max) * (SPARK.length - 1)))]!, n: c }))
}

const tokens = (n: number) => (n >= 1_000_000 ? `${(n / 1_000_000).toFixed(1)}M` : n >= 1000 ? `${Math.round(n / 1000)}k` : `${n}`)
const DREAM = { done: ['success', '✔ completato'], running: ['warning', '◐ in corso'], paused: ['warning', '‖ in pausa'], failed: ['error', '✖ fallito'] } as const

/** The state of the last alignment: colour and words. */
function syncText(p: Panel, now: number): [string, string] {
  if (!p.git) return ['inactive', 'fuori da git']
  const s = p.sync
  if (!s) return ['inactive', 'al prossimo commit']
  if (s.state === 'running') return ['warning', '◐ allineamento']
  if (s.state === 'failed') return ['error', `✖ ${s.error ?? 'non riuscito'}`]
  return ['success', `✔ ${ago(now - s.at)} +${s.uploaded} −${s.removed}${s.skipped ? ` ⊘${s.skipped}` : ''}`]
}

function monitor(ui: Ui, p: Panel, now: number, o: { hide: () => void; refresh: () => void }) {
  const { Box, Text, Button } = ui
  if (p.problem)
    return (
      <Box borderStyle="round" borderColor="error" paddingX={1}>
        <Text>
          <Text weight="bold" color="claude">◆ Company Brain </Text>
          <Text color="error">● </Text>
          <Text dimColor>non disponibile: {p.problem}</Text>
        </Text>
      </Box>
    )
  const recent = p.flows.filter((f) => now - f.at < RATE_WINDOW_MS)
  const of = (xs: Flow[], dir: Flow['dir']) => xs.filter((f) => f.dir === dir)
  const bytes = (xs: Flow[]) => xs.reduce((n, f) => n + f.bytes, 0)
  const s = p.status
  const src = s?.sources ?? {}
  const done = src.compiled ?? 0
  const failed = src.failed ?? 0
  const working = Object.entries(src).filter(([k]) => k !== 'compiled' && k !== 'failed').reduce((n, [, v]) => n + v, 0)
  const docs = done + failed + working
  // the link: green when the state is fresh, amber when old or the last refresh failed
  const fresh = p.statusAt !== null && now - p.statusAt < 3 * STATUS_EVERY_MS && !p.statusError
  const link = s ? (fresh ? 'success' : 'warning') : p.statusError ? 'error' : 'inactive'
  const segments = [
    { n: done, color: 'success', label: 'compilati' },
    { n: working, color: 'warning', label: 'in lavorazione' },
    { n: failed, color: 'error', label: 'falliti' },
  ].filter((x) => x.n > 0)
  const grow = (n: number) => Math.max(1, Math.round((n / Math.max(1, docs)) * 1000))
  const io = (dir: Flow['dir'], arrow: string, label: string, color: string) => (
    <Text>
      <Text color={color}>{arrow} </Text>
      <Text>{label} </Text>
      <Text dimColor tabular>
        {perMin(of(recent, dir).length)}/min {kb(bytes(of(recent, dir)))}{' '}
      </Text>
      {/* a quiet minute is a dim floor, a busy one a coloured bar */}
      {sparkline(p.flows, dir, now).map((b) => (
        <Text color={b.n ? color : 'inactive'}>{b.glyph}</Text>
      ))}
    </Text>
  )
  const [syncColor, syncWords] = syncText(p, now)
  const files = (
    <Text wrap="truncate">
      <Text weight="bold">{p.slug} </Text>
      <Text dimColor tabular>{p.files} file </Text>
      <Text color={syncColor}>{syncWords}</Text>
    </Text>
  )
  const when = <Text dimColor tabular>{p.statusAt ? ago(now - p.statusAt) : p.statusError ? 'offline' : '…'}</Text>
  return (
    <Box flexDirection="column" borderStyle="round" borderColor="subtle" paddingX={1}>
      <Box flexDirection="row" alignItems="center" columnGap={1}>
        <Box flexShrink={0} flexDirection="row">
          <Text>
            <Text weight="bold" color="claude">◆ Company Brain </Text>
            <Text color={link}>● </Text>
          </Text>
          {files}
        </Box>
        <Box flexGrow={1} width={0} />
        <Box flexShrink={0} flexDirection="row" alignItems="center" columnGap={1}>
          {when}
          <Button key="refresh" label="↻" onPress={o.refresh} />
          <Button key="hide" label="Hide" onPress={o.hide} />
        </Box>
      </Box>
      <Box flexDirection="row" flexWrap="wrap" columnGap={2}>
        <Text weight="bold">I/O</Text>
        {io('write', '▲', 'in', 'suggestion')}
        {io('read', '▼', 'out', 'claude')}
        <Text dimColor tabular>
          Σ {of(p.flows, 'read').length}r/{of(p.flows, 'write').length}w
        </Text>
      </Box>
      {s ? (
        <Box flexDirection="column">
          <Box flexDirection="row" alignItems="center" columnGap={1}>
            <Box flexShrink={0}>
              <Text>
                <Text weight="bold">Vault </Text>
                <Text dimColor tabular>
                  {s.pages.pages} pag · {s.pages.inbox} inbox · {tokens(s.pages.tokens)} tok · {docs} doc
                </Text>
              </Text>
            </Box>
            {/* Width 0 plus flexGrow by count: the bar takes the room left, split by share. */}
            <Box flexDirection="row" alignItems="center" flexGrow={1} width={0} height={1} overflow="hidden">
              {segments.length ? (
                segments.map((x) => <Box flexGrow={grow(x.n)} width={0} minWidth={1} height="50%" backgroundColor={x.color} />)
              ) : (
                <Box flexGrow={1} width={0} height="30%" backgroundColor="subtle" />
              )}
            </Box>
          </Box>
          <Box flexDirection="row" flexWrap="wrap" columnGap={2}>
            {[
              { n: done, color: 'success', label: 'compilati' },
              { n: working, color: 'warning', label: 'in lavorazione' },
              { n: failed, color: 'error', label: 'falliti' },
            ].map((x) => (
              <Text>
                <Text color={x.n ? x.color : 'inactive'}>■</Text>
                <Text> {x.label} </Text>
                <Text dimColor tabular>{x.n}</Text>
              </Text>
            ))}
            <Text>
              <Text>24h </Text>
              <Text dimColor tabular>+{s.sourcesLast24h}</Text>
            </Text>
          </Box>
          <Box flexDirection="row" flexWrap="wrap" columnGap={2}>
            <Text>
              <Text weight="bold">Coda </Text>
              <Text color={s.work.running ? 'warning' : 'inactive'}>● </Text>
              <Text dimColor tabular>{s.work.running} run </Text>
              <Text color={s.work.queued ? 'suggestion' : 'inactive'}>○ </Text>
              <Text dimColor tabular>{s.work.queued} wait </Text>
              <Text color={s.work.failed ? 'error' : 'inactive'}>✖ </Text>
              <Text dimColor tabular>{s.work.failed} fail</Text>
            </Text>
            <Text>
              <Text weight="bold">Notte </Text>
              {s.dream ? (
                <Text>
                  <Text color={DREAM[s.dream.status][0]}>{DREAM[s.dream.status][1]}</Text>
                  <Text dimColor tabular>
                    {' '}
                    {s.dream.night}
                    {s.dream.status === 'running' && s.dream.step ? ` · ${s.dream.step}` : ''}
                  </Text>
                </Text>
              ) : (
                <Text dimColor>mai eseguito</Text>
              )}
            </Text>
          </Box>
        </Box>
      ) : (
        <Text dimColor>{p.statusError ? `Stato del vault non disponibile: ${p.statusError}` : 'Lettura dello stato del vault…'}</Text>
      )}
      {s && p.statusError ? <Text color="warning">Ultimo aggiornamento non riuscito: {p.statusError}</Text> : null}
    </Box>
  )
}

function parse(text: string): any {
  try {
    return JSON.parse(text)
  } catch {
    return null
  }
}
````

### `hooks/register.test.ts`

````ts
// The mod against a pretend server: git and SSH (curl on the server) answered by the test.
import { describe, expect, mock, test } from 'claude-code/testing'

/** The settings of the tests: the server answers over SSH, as curl on the server. */
const OPTIONS = { options: { url: 'https://brain.test:3700', sshHost: 'utente@brain.test' } }

type Call = { method: string; path: string; body?: any }

/** The server of the test: the project routes of Company Brain, in memory. */
function server(o: { created?: boolean } = {}) {
  const calls: Call[] = []
  let created = o.created ?? false
  /** Repository files of the vault: path → commit and content. */
  const files = new Map<string, { commit?: string; content: string }>()
  function answer(config: string): { status: number; body: unknown } {
    const url = /url = "([^"]+)"/.exec(config)![1]!
    const method = /request = "([^"]+)"/.exec(config)![1]!
    const raw = /data-binary = "(.*)"$/m.exec(config)?.[1]
    const body = raw ? JSON.parse(raw.replace(/\\"/g, '"').replace(/\\\\/g, '\\')) : undefined
    const path = url.replace(/^https:\/\/127\.0\.0\.1:3700/, '')
    calls.push({ method, path, body })
    expect(config).toContain('Authorization: Bearer chiave-di-prova')
    if (method === 'GET' && path === '/api/projects/mio-progetto') {
      if (!created) return { status: 404, body: { error: { message: 'Progetto non trovato.' } } }
      return { status: 200, body: { vault: { id: 'vlt_1', slug: 'mio-progetto', name: 'Mio-Progetto' }, sections: [] } }
    }
    if (method === 'POST' && path === '/api/projects') return (created = true), { status: 201, body: { id: 'vlt_1', created: true } }
    if (method === 'GET' && path === '/api/projects/mio-progetto/status')
      return {
        status: 200,
        body: {
          vault: { id: 'vlt_1', slug: 'mio-progetto', name: 'Mio-Progetto' },
          pages: { pages: 42, inbox: 3, sections: 0, tokens: 12345, updatedAt: '2026-11-10T10:00:00Z' },
          sources: { compiled: 7, indexed: 2, failed: 1 },
          sourcesLast24h: 4,
          work: { queued: 2, running: 1, failed: 0 },
          dream: { night: '2026-11-10', status: 'done', step: null, startedAt: '2026-11-10T22:00:00Z', finishedAt: '2026-11-10T22:30:00Z', tokens: 900 },
        },
      }
    if (method === 'GET' && path === '/api/projects/mio-progetto/files')
      return { status: 200, body: { files: [...files].map(([p, f], i) => ({ path: p, sourceId: `src_${i}`, status: 'compiled', ...(f.commit ? { commit: f.commit } : {}) })) } }
    if (method === 'PUT' && path === '/api/projects/mio-progetto/files') {
      const cur = files.get(body.path)
      const content = atob(body.content)
      files.set(body.path, { commit: body.commit, content })
      return { status: 200, body: { path: body.path, created: cur?.content !== content, removed: cur && cur.content !== content ? 1 : 0 } }
    }
    const del = /^\/api\/projects\/mio-progetto\/files\?path=(.+)$/.exec(path)
    if (method === 'DELETE' && del) {
      const had = files.delete(decodeURIComponent(del[1]!))
      return { status: 200, body: { removed: had ? 1 : 0 } }
    }
    return { status: 500, body: {} }
  }
  return { calls, files, answer }
}

/** A stand-in for a call on $: what the call resolves to. */
const value = (v: unknown) => ({ value: v }) as never
const ran = (exitCode: number, stdout: string, stderr = '') => value({ exitCode, stdout, stderr, isStdoutTruncated: false, isStderrTruncated: false })

const BASE_COMPOSE = { model: 'm', promptModel: 'm', surfaces: [], tools: [], outputStyle: null, traits: [] }
const ROOT = 'D:/lavoro/mio-progetto'
const HEAD = 'c0ffee0000000000000000000000000000000002'
const OLD = 'c0ffee0000000000000000000000000000000001'
/** A commit made during the session. */
const NEXT = 'c0ffee0000000000000000000000000000000003'
const norm = (p: unknown) => String(p).replaceAll('\\', '/')

/** The repository of the test; `ls` is what vault-docs.conf matches, `diff` what changed since OLD. */
function repo(o: { head: () => string; ls: () => string[]; diff?: string[] }) {
  return (argv: string[]) => {
    const args = argv.slice(1).join(' ')
    if (args === 'rev-parse --show-toplevel') return ran(0, `${ROOT}\n`)
    if (args === 'remote get-url origin') return ran(0, 'git@github.com:acme/Mio-Progetto.git\n')
    if (args === 'rev-parse HEAD') return ran(0, `${o.head()}\n`)
    if (args === `ls-files -z --cached --ignored --exclude-from=${ROOT}/vault-docs.conf`) return ran(0, o.ls().map((f) => `${f}\0`).join(''))
    if (args.startsWith(`diff --name-only ${OLD} `)) return ran(0, (o.diff ?? []).join('\n'))
    if (args.startsWith('diff --name-only ')) return ran(128, '', 'bad revision')
    return ran(1, '', `unexpected: git ${args}`)
  }
}

/** The engine around the mod: files of the repository, the server over SSH, the status line. */
function engine(on: any, srv: ReturnType<typeof server>, git: (argv: string[]) => never, disk: Map<string, string>, o: { tools?: boolean } = {}) {
  const status: (string | undefined)[] = []
  const toasts: string[] = []
  on('session.root', async () => value(ROOT))
  on('env.get', async () => value('C:/Users/prova'))
  on('fs.read', async (_$: unknown, e: { path: string; as?: string }) => {
    const p = norm(e.path)
    if (p.endsWith('/company-brain-progetti.key')) return value('chiave-di-prova\n')
    const text = disk.get(p.replace(`${ROOT}/`, ''))
    if (text === undefined) throw new Error(`ENOENT ${p}`)
    return value(e.as === 'bytes' ? { base64: btoa(text) } : text)
  })
  on('fs.write', async (_$: unknown, e: { path: string; text: string }) => (disk.set(norm(e.path).replace(`${ROOT}/`, ''), e.text), value(undefined)))
  on('fs.exists', async (_$: unknown, e: { path: string }) => value(disk.has(norm(e.path).replace(`${ROOT}/`, ''))))
  on('fs.stat', async (_$: unknown, e: { path: string }) => {
    const text = disk.get(norm(e.path).replace(`${ROOT}/`, ''))
    if (text === undefined) throw new Error('ENOENT')
    return value({ kind: 'file', size: text.length, mtimeMs: 0, isLink: false })
  })
  on('process.run', async (_$: unknown, e: { argv: string[]; init?: { stdin?: string } }) => {
    if (e.argv[0] === 'git') return git(e.argv)
    expect(e.argv.slice(0, 5)).toEqual(['ssh', '-o', 'BatchMode=yes', '-o', 'ConnectTimeout=10'])
    // the key is never on the command line
    expect(e.argv.join(' ')).not.toContain('chiave-di-prova')
    const r = srv.answer(String(e.init?.stdin))
    return ran(0, `${JSON.stringify(r.body)}\n${r.status}`)
  })
  on('command.register', async () => value(undefined))
  on('ui.status', async (_$: unknown, e: { text?: string }) => (status.push(e.text), value(undefined)))
  on('ui.toast', async (_$: unknown, e: { text: string }) => (toasts.push(e.text), value(undefined)))
  on('session.start', async (_$: unknown, e: { cwd: string }) => ({ cwd: e.cwd }))
  on('prompt.compose', async () => ({ sections: [{ id: 'intro', text: 'base', scope: 'shared' as const }] }))
  on('turn.complete', async () => ({ text: 'fatto' }))
  if (!o.tools) on('tool.call', async () => ({ result: 'ok' }))
  on('prompt.submit', async (_$: unknown, e: { text: string; context?: string[] }) => ({ text: e.text, context: e.context }) as never)
  return { status, toasts }
}

const turn = (id: string, extra: object = {}) => ({ answer: 'fatto', durationMs: 1, isAborted: false, turnId: id, reason: 'answer', ...extra }) as never
const routes = (calls: Call[]) => calls.map((c) => `${c.method} ${c.path}`)
/** The calls but the refresh of the band, which runs on its own timer. */
const work = (calls: Call[]) => calls.filter((c) => !c.path.endsWith('/status'))

describe('company-brain-progetti', () => {
  test('creates the vault and vault-docs.conf, keeps an existing conf, says so in the prompt', OPTIONS, async ($, on) => {
    const srv = server()
    mock.clock(on)
    const disk = new Map<string, string>()
    const { status } = engine(on, srv, repo({ head: () => HEAD, ls: () => [] }) as never, disk)
    await $.session.start({ cwd: ROOT } as never)
    expect(routes(srv.calls).slice(0, 4)).toEqual(['GET /api/projects/mio-progetto', 'POST /api/projects', 'GET /api/projects/mio-progetto', 'GET /api/projects/mio-progetto/files'])
    expect(srv.calls[1]!.body).toEqual({ slug: 'mio-progetto', name: 'Mio-Progetto' })
    const conf = disk.get('vault-docs.conf')!
    expect(conf).toContain('come in .gitignore')
    expect(conf.split('\n').filter((l) => l && !l.startsWith('#'))).toEqual(['README.md', 'docs/**/*.md'])
    expect(status.at(-1)).toBe('Company Brain · Mio-Progetto: 0 file da vault-docs.conf')

    const prompt = await $.prompt.compose(BASE_COMPOSE as never)
    const docs = prompt.sections.find((s) => s.id === 'company-brain-progetti:docs')!
    expect(docs.scope).toBe('session')
    expect(docs.text).toContain('vault `mio-progetto`')
    expect(docs.text).toContain('elencati in `vault-docs.conf`')
    expect(docs.text).not.toContain('project_doc')

    // the person's conf is never overwritten
    disk.set('vault-docs.conf', 'docs/\n')
    await $.session.start({ cwd: ROOT } as never)
    expect(disk.get('vault-docs.conf')).toBe('docs/\n')
  })

  test('a new commit aligns the vault: missing and changed files up, files no longer named down', OPTIONS, async ($, on) => {
    const srv = server({ created: true })
    srv.files.set('README.md', { commit: OLD, content: 'vecchio' })
    srv.files.set('docs/a.md', { commit: OLD, content: 'A' })
    srv.files.set('docs/tolto.md', { commit: OLD, content: 'via' })
    const clock = mock.clock(on)
    let head = HEAD
    const disk = new Map([
      ['vault-docs.conf', 'README.md\ndocs/\n'],
      ['README.md', 'nuovo'],
      ['docs/a.md', 'A'],
      ['docs/b.md', 'B'],
    ])
    const { status, toasts } = engine(on, srv, repo({ head: () => head, ls: () => ['README.md', 'docs/a.md', 'docs/b.md'], diff: ['README.md', 'src/main.rs'] }) as never, disk)
    await $.session.start({ cwd: ROOT } as never)
    srv.calls.length = 0

    // the first prompt carries the hidden context, the second does not
    const first = await $.prompt.submit({ text: 'ciao', origin: { kind: 'composer' }, wait: false } as never)
    expect(first.context?.[0]).toContain('vaults ["mio-progetto"]')
    expect((await $.prompt.submit({ text: 'ancora', origin: { kind: 'composer' }, wait: false } as never)).context).toBeUndefined()

    // a turn without a commit: nothing
    await $.turn.complete(turn('t1'))
    await clock.advance(1)
    expect(work(srv.calls)).toEqual([])

    // a commit: README changed, docs/b.md is new, docs/a.md is the same, docs/tolto.md no longer named
    head = NEXT
    await $.turn.complete(turn('t2'))
    await clock.advance(1)
    const writes = work(srv.calls).filter((c) => c.method !== 'GET')
    expect(writes.map((c) => `${c.method} ${c.body?.path ?? c.path}`)).toEqual(['PUT README.md', 'PUT docs/b.md', 'DELETE /api/projects/mio-progetto/files?path=docs%2Ftolto.md'])
    expect(writes[0]!.body.commit).toBe(NEXT)
    expect(srv.files.get('README.md')!.content).toBe('nuovo')
    expect([...srv.files.keys()].sort()).toEqual(['README.md', 'docs/a.md', 'docs/b.md'])
    expect(toasts.at(-1)).toBe('Company Brain: vault «mio-progetto» allineato (2 caricati, 2 tolti).')
    expect(status.at(-1)).toBe('Company Brain · Mio-Progetto: 3 file da vault-docs.conf')

    // the same commit is not aligned twice; an interrupted turn and a subagent's turn do nothing
    srv.calls.length = 0
    await $.turn.complete(turn('t3'))
    head = 'c0ffee0000000000000000000000000000000004'
    await $.turn.complete(turn('t4', { reason: 'aborted' }))
    await $.turn.complete(turn('t5', { agentId: 'a1' }))
    await clock.advance(1)
    expect(work(srv.calls)).toEqual([])
  })

  test('a failed alignment is tried again at the next turn', OPTIONS, async ($, on) => {
    const srv = server({ created: true })
    const clock = mock.clock(on)
    let head = HEAD
    const disk = new Map([['vault-docs.conf', 'README.md\n'], ['README.md', 'testo']])
    let fail = true
    const git = repo({ head: () => head, ls: () => ['README.md'] })
    const { status } = engine(on, srv, ((argv: string[]) => (fail && argv[1] === 'ls-files' ? ran(128, '', 'fatal: boom') : git(argv))) as never, disk)
    await $.session.start({ cwd: ROOT } as never)
    head = NEXT
    await $.turn.complete(turn('t1'))
    await clock.advance(1)
    expect(status.at(-1)).toContain('allineamento non riuscito (git ls-files: fatal: boom)')
    fail = false
    await $.turn.complete(turn('t2'))
    await clock.advance(1)
    expect([...srv.files.keys()]).toEqual(['README.md'])
  })

  test('a server without the file routes keeps the vault and says why there is no alignment', OPTIONS, async ($, on) => {
    const srv = server({ created: true })
    const answer = srv.answer
    srv.answer = (config: string) => (config.includes('/files') ? { status: 404, body: { message: 'Route GET:/api/projects/mio-progetto/files not found', error: 'Not Found', statusCode: 404 } } : answer(config))
    mock.clock(on)
    const { status } = engine(on, srv, repo({ head: () => HEAD, ls: () => [] }) as never, new Map([['vault-docs.conf', 'README.md\n']]))
    await $.session.start({ cwd: ROOT } as never)
    expect(status.at(-1)).toContain('Company Brain · Mio-Progetto: 0 file da vault-docs.conf · allineamento non riuscito (il server non ha ancora i file di progetto')
  })

  test('a folder outside git gets its conf but no alignment', OPTIONS, async ($, on) => {
    const srv = server()
    const clock = mock.clock(on)
    const disk = new Map([['Cargo.toml', '[package]']])
    engine(on, srv, (() => ran(128, '', 'not a git repository')) as never, disk)
    await $.session.start({ cwd: ROOT } as never)
    expect(disk.has('vault-docs.conf')).toBe(true)
    await $.turn.complete(turn('t1'))
    await clock.advance(1)
    expect(srv.calls.some((c) => c.method === 'PUT')).toBe(false)
    const prompt = await $.prompt.compose(BASE_COMPOSE as never)
    expect(prompt.sections.at(-1)!.text).toContain('fuori da git la mod non li allinea')
  })

  test('outside a project the mod stays silent', OPTIONS, async ($, on) => {
    on('session.root', async () => value('C:/Users/prova'))
    on('env.get', async () => value('C:/Users/prova'))
    on('process.run', async () => ran(128, '', 'not a git repository'))
    on('command.register', async () => value(undefined))
    on('ui.status', async () => value(undefined))
    on('session.start', async (_$, e) => ({ cwd: (e as { cwd: string }).cwd }))
    on('prompt.compose', async () => ({ sections: [{ id: 'intro', text: 'base', scope: 'shared' as const }] }))
    await $.session.start({ cwd: 'C:/Users/prova' } as never)
    const prompt = await $.prompt.compose(BASE_COMPOSE as never)
    expect(prompt.sections.map((s) => s.id)).toEqual(['intro'])
  })

  test('the band shows the vault, its files, the server state and the reads and writes; no pane', OPTIONS, async ($, on) => {
    const srv = server({ created: true })
    srv.files.set('README.md', { commit: HEAD, content: 'x' })
    mock.clock(on)
    const disk = new Map([['vault-docs.conf', 'README.md\n']])
    const opened: string[] = []
    on('ui.open', async (_$, e) => (opened.push((e as { id: string }).id), value({ isPlaced: true })))
    engine(on, srv, repo({ head: () => HEAD, ls: () => ['README.md'] }) as never, disk, { tools: true })
    on('tool.call', async (_$, e) => ({ result: (e as { tool: string }).tool.endsWith('brain_search') ? 'x'.repeat(2048) : 'ok' }))
    // the engine beneath (and context-bar, in a session): it draws "sotto"
    on('ui.render', (_$, e) => {
      const { Text } = _$.ui.resolve(e as never) as { Text: unknown }
      return (globalThis as unknown as { h: (...a: unknown[]) => never }).h(Text, {}, 'sotto')
    })
    await $.session.start({ cwd: ROOT, surface: 'desktop', isInteractive: true } as never)
    expect(opened).toEqual([])

    await $.tool.call({ tool: 'mcp__plugin_company-brain-progetti_docs__brain_search', query: 'octree' } as never)
    await $.tool.call({ tool: 'mcp__plugin_company-brain-progetti_docs__brain_read', id: 'x' } as never)
    await $.tool.call({ tool: 'mcp__plugin_company-brain-progetti_docs__brain_capture', title: 'Nota', content: 'Deciso: LOD.' } as never)
    await $.tool.call({ tool: 'mcp__altro__cerca', q: 'x' } as never)

    const BAND = { plugin: 'company-brain-progetti', component: 'AbovePrompt', props: { hasSurvey: false, isWorking: false, maxRows: 20, bodyColumns: 100, scroll: { offset: 0, bodyRows: 20 }, view: {} } }
    for (const surface of ['terminal', 'desktop'] as const) {
      const ui = await $.ui.mount({ ...BAND, surface } as never)
      const has = async (re: RegExp) => expect(await ui.find({ type: 'Text', text: re } as never)).toBeDefined()
      await has(/◆ Company Brain/)
      await has(/^mio-progetto $/)
      await has(/^1 file $/)
      await has(/al prossimo commit/)
      await has(/0,2\/min \d+ B/) // in: one write
      await has(/0,4\/min 2\.0 kB/) // out: two reads, 2 kB of search result
      await has(/Σ 2r\/1w/)
      await has(/42 pag · 3 inbox · 12k tok · 10 doc/)
      await has(/✔ completato/)
      await has(/^sotto$/)
      await ui.unmount()
    }

    // Hide gives the place back to what is beneath; /progetto-vault shows it again
    const band = await $.ui.mount({ ...BAND, surface: 'desktop' } as never)
    await band.press({ key: 'hide' } as never)
    await band.unmount()
    const after = await $.ui.mount({ ...BAND, surface: 'desktop' } as never)
    expect(await after.find({ type: 'Text', text: /Company Brain/ } as never)).toBeUndefined()
    await after.unmount()
    expect((await $.command.run({ command: 'progetto-vault', args: '' } as never)).text).toBe('Monitor del vault «mio-progetto» sopra il prompt.')
    const again = await $.ui.mount({ ...BAND, surface: 'desktop' } as never)
    expect(await again.find({ type: 'Text', text: /Company Brain/ } as never)).toBeDefined()
    await again.unmount()
  })
})
````

### `README.md`

````markdown
# company-brain-progetti

Un vault di Company Brain per ogni progetto che Claude Code segue (un repository git, oppure una
cartella con un file di progetto come `Cargo.toml` o `package.json`). I file di documentazione del
repository sono documenti del vault, allineati a ogni commit.

- **vault-docs.conf:** all'inizio della sessione la mod trova il progetto (nome del remote
  `origin`, altrimenti la cartella), crea il suo vault la prima volta e crea `vault-docs.conf`
  nella radice del progetto se non c'è (con `README.md` e `docs/**/*.md`). Il file elenca i
  documenti, una regola per riga, come in `.gitignore`: file singoli, cartelle (`docs/`),
  wildcard (`*.md`, `docs/**/*.pdf`), `!` per escludere, `#` per i commenti. git confronta le
  regole con i file che traccia (`git ls-files --cached --ignored --exclude-from`).
- **Allineamento, solo al commit, in background:** alla fine di ogni turno la mod confronta
  `HEAD` con il commit visto prima. Se c'è un nuovo commit allinea il vault con i file che
  `vault-docs.conf` indica e che esistono: carica i file che il vault non ha e quelli cambiati
  dal commit a cui il vault li ha visti, cancella dal vault i file non più indicati o non più nel
  repository. Il server tiene un documento per percorso: una nuova versione sostituisce la
  vecchia. Company Brain compila i documenti in pagine (ingestion). Un file oltre 4 MiB è
  saltato. Un allineamento non riuscito riprova alla fine del turno successivo. Fuori da git non
  c'è allineamento.
- **Memoria di Company Brain (MCP):** Claude usa gli strumenti di Company Brain
  (`brain_overview`, `brain_search`, `brain_read`, `brain_capture`, …) e le risorse
  `brain://<vault>/llms.txt`: legge il vault del progetto, cerca analisi e concetti negli altri
  vault di progetto che la persona può leggere, e scrive note nella inbox del vault del
  progetto. Il primo prompt della sessione porta, nascosto, l'ordine di cercare nel vault prima
  di lavorare.
- **Monitor:** una barra sopra il prompt (come context-bar, e sopra di essa) mostra il vault:
  ● stato del collegamento (verde aggiornato, giallo vecchio o non riuscito, rosso offline), i
  file nel vault e l'ultimo allineamento (✔ quando, +caricati −tolti ⊘saltati), I/O con ▲
  scritture nel vault e ▼ letture dal vault al minuto (ultimi 5 minuti), byte e una sparkline
  degli ultimi 12 minuti, pagine, note in inbox, token, documenti con una barra per stato
  (compilati, in lavorazione, falliti), coda dei task e ultimo ciclo notturno. Si aggiorna ogni
  minuto e dopo ogni scrittura; ↻ la aggiorna subito, «Hide» la nasconde e `/progetto-vault` la
  mostra di nuovo.

## Strumenti

Il server MCP stdio della mod (`server/tools.mjs`, avviato da `.mcp.json`; serve `node` nel
PATH) è un proxy verso `POST /mcp` di Company Brain: elenca gli strumenti, le risorse e i prompt
del server. Legge le impostazioni da `~/.claude/company-brain-progetti.json`, che gli hook
scrivono all'inizio di ogni sessione. Test del proxy: `node --test server/tools.test.mjs`. Test
degli hook: `claude plugin test .`.

## Server

L'allineamento usa le route `GET`, `PUT` e `DELETE /api/projects/<slug>/files` di Company Brain.
Un server senza queste route risponde 404: la mod dice che il server è di una versione
precedente.

## Impostazioni

- `url`: indirizzo del server (predefinito `https://tuo-server:3700`).
- `sshHost`: `utente@host` del server. Le richieste passano da SSH e `curl` sul server; la
  chiave viaggia su stdin. Vuoto: HTTPS diretto.
- `caFile`: con `sshHost` vuoto, il certificato PEM della CA del server (per esempio
  `data/tls/ca.pem` del repository di Company Brain). Le richieste usano `curl` e il proxy
  Node con questa CA. Serve una rete che non intercetta l'HTTPS verso il server (per Avast:
  un'eccezione per l'indirizzo del server).
- `apiKeyFile`: file con la chiave di un account API con gli ambiti `read` e `projects`, legato
  alla persona che usa questa installazione di Claude Code.

Crea l'account sul server, legato alla persona:

```bash
npm run -s cb -- api create claude-code-nome.cognome --scopes read,projects --user nome.cognome > chiave.txt
```

Poi copia il file in `~/.claude/company-brain-progetti.key`. La chiave legge solo i vault che la
persona può leggere (mai sopra «interno», mai i suoi vault personali) e scrive solo nei vault di
progetto su cui la persona ha la scrittura. Una chiave con il solo ambito `projects` allinea i
file ma non usa MCP.
````
