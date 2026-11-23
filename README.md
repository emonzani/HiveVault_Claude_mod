# HiveVault Claude mod

Esempio d'uso di una mod di Claude Code: `company-brain-progetti`. Collega Claude Code a un server
Company Brain self-hosted e gli dà la memoria di progetto.

Il repository non contiene la mod già pronta. Contiene [INSTALL_FOR_CLAUDE.md](INSTALL_FOR_CLAUDE.md),
un file di istruzioni che Claude Code segue per ricreare la mod sul tuo computer, installarla,
configurarla e verificarla.

## Cosa fa la mod

- **Un vault per progetto.** In ogni repository git (o cartella con un file di progetto come
  `package.json`) la mod usa un vault di Company Brain e lo crea alla prima sessione.
- **Documentazione allineata a ogni commit.** I file elencati in `vault-docs.conf` (sintassi di
  `.gitignore`) vengono caricati nel vault in background. I file nuovi o cambiati salgono, quelli
  tolti spariscono.
- **Strumenti MCP.** Un proxy stdio verso `POST /mcp` del server dà a Claude gli strumenti
  `brain_*`: cercare, leggere e scrivere note nel vault del progetto e negli altri vault leggibili.
- **Barra sopra il prompt.** Mostra lo stato del vault, l'ultimo allineamento e le letture e
  scritture della sessione. `/progetto-vault` la mostra o la nasconde.

## Cosa ti serve

- Claude Code
- `node`, `git` e `curl` nel PATH
- `ssh`, se raggiungi il server via SSH
- un server Company Brain raggiungibile
- una chiave API del server con gli ambiti `read` e `projects`, salvata in un file (predefinito
  `~/.claude/company-brain-progetti.key`)

## Installazione

1. Scarica il file di istruzioni:

   ```bash
   curl -fsSLO https://raw.githubusercontent.com/emonzani/HiveVault_Claude_mod/main/INSTALL_FOR_CLAUDE.md
   ```

2. Apri Claude Code nella stessa cartella e scrivi:

   ```
   Segui le istruzioni di INSTALL_FOR_CLAUDE.md per installare la mod company-brain-progetti.
   ```

3. Rispondi alle domande di Claude: indirizzo del server, modalità di accesso (HTTPS, HTTPS con il
   file della CA, SSH) e percorso del file della chiave. Claude non legge e non stampa la chiave.

4. Apri una nuova sessione di Claude Code in un progetto git. Se tutto funziona vedi la barra
   «◆ Company Brain» sopra il prompt e il file `vault-docs.conf` nella radice del progetto.

I problemi comuni e la disinstallazione sono in fondo a [INSTALL_FOR_CLAUDE.md](INSTALL_FOR_CLAUDE.md).

## Sicurezza

- La chiave API resta nel tuo file e non entra mai nei file della mod. Con SSH viaggia su stdin, mai
  sulla riga di comando.
- La chiave vede solo i vault che la persona a cui è legata può leggere.
- Leggi `INSTALL_FOR_CLAUDE.md` prima di passarlo a Claude: contiene il codice che verrà scritto
  sul tuo computer.
