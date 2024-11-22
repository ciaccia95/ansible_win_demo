# Windows demo

Questo playbook di Ansible permette di creare e configurare una Virtual Machine Windows su Microsoft Azure, utilizzando un approccio IaaC. Include la configurazione di rete, sicurezza, abilitazione di WinRM e l'installazione di applicazioni e servizi su Windows. Inoltre, utilizza l'inventory dinamico di Azure per gestire le risorse.

## Prerequisiti

1. Accedere ad azure con il comando `az login`
1. Assicurarsi di avere la azure collection

   ```bash
   ansible-galaxy collection install azure.azcollection
   ```

## Struttura del playbook

### Play 1: Creazione della VM su Azure

#### Tasks Principali

- Creazione della rete virtuale (`azure_rm_virtualnetwork`)
  Configura la `VNet` con uno specifico indirizzo IP.
- Aggiunta di una subnet (`azure_rm_subnet`)
  Crea una subnet all'interno della `VNet`.
- Creazione dell'indirizzo IP pubblico (`azure_rm_publicipaddress`)
  Configura un indirizzo IP statico per la VM.
- Creazione del Network Security Group (`azure_rm_securitygroup`)
  Definisce regole di sicurezza per `RDP`, `WinRM` e traffico web in `HTTP/HTTPS`.
- Creazione della NIC (`azure_rm_networkinterface`)
  Configura la scheda di rete associata alla `VNet` e al `NSG` (Network Security Group).
- Creazione della VM (`azure_rm_virtualmachine`):

  - Specifica l'immagine Windows Server 2019 Datacenter.
  - Configura credenziali di accesso.
  - Associa la NIC configurata.
  - Abilitazione del listener `HTTPS` per `WinRM`:

- Estensione `CustomScriptExtension` per eseguire uno script PowerShell da remoto.
  Script scaricato dal repository GitHub ufficiale.
- Attesa della disponibilità di `WinRM`
  Il task verifica che il servizio `WinRM` sia raggiungibile sulla porta `5986`.

### Play 2: Configurazione della VM Windows

#### Tasks principali

- Installazione di `IIS` (`win_feature`)
  Abilita il ruolo di Web Server.
- Rimozione del sito web predefinito (`win_iis_website`)
  Rimuove il sito predefinito di `IIS`.
- Creazione di una directory per il nuovo sito (`win_file`)
  Configura la directory `C:\inetpub\wwwroot\demo`.
- Creazione di una pagina web (`win_template`)
  Genera un file `HTML` usando un template `Jinja2`.
- Configurazione del nuovo sito `IIS` (`win_iis_website`)
  Aggiunge un sito denominato `DemoSite` su `IIS`.
- Installazione pacchetto `Notepad++`
  - Crea una folder per file temporanei (`win_file`) `C:\temp`
  - Scarica l'installer di `Notepad++` (`win_get_url`).
  - Installa `Notepad++` in modalità silenziosa (win_package).
  - Rimuove la directory temporanea.

## Configurazione dell'Inventory Dinamico

L'inventory dinamico di Azure consente di gestire automaticamente le risorse su Azure.

### Installazione del plugin `azure_rm`

Installo con il comando

```bash
ansible-galaxy collection install azure.azcollection
```

### Configurazione del file azure_rm.yml

Questo file definisce le credenziali e i filtri per recuperare le risorse

```yaml
plugin: azure_rm
include_vm_resource_groups:
  - RESOURCE_GROUP_NAME # <-- DA POPOLARE
auth_source: auto
conditional_groups:
  windows: "'WindowsServer' in image.offer"
```

Per ulteriori dettagli: [Azure Dynamic Inventory](https://learn.microsoft.com/en-us/azure/developer/ansible/dynamic-inventory-configure?tabs=azure-cli).

Verifica dell'inventory:

```bash
ansible-inventory --graph -i azure_rm.yml
```

## Configurazione di WinRM su Windows

WinRM è il protocollo utilizzato per comunicare con le macchine Windows.

### Script PowerShell

Lo script `ConfigureRemotingForAnsible.ps1` scaricato dal repository di Ansible abilita e configura WinRM per le connessioni remote.

Comando da eseguire manualmente:

```powershell
powershell -ExecutionPolicy Unrestricted -File ConfigureRemotingForAnsible.ps1
```

Per ulteriori informazioni: [Windows Configuration](https://learn.microsoft.com/en-us/azure/developer/ansible/vm-configure-windows?tabs=ansible).

> **Per far funzionare tutto su MacOS**
>
> Assicurati di avere il venv corretto
>
> ```bash
> conda activate ansible
> cd win-demo
> export export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
> ```
>
> Assicurati di
>
> ```bash
> ansible-playbook playbook.yaml -e resource_group_name="VALORE"
> ```
>
> Assicurati anche di **NOTA CHE L'INVENTARIO DEVE CHIAMARSI `azure_rm.yml`**
>
> ```diff
> plugin: azure_rm
> include_vm_resource_groups:
> -  - VALORE VECCHIO
> +  - VALORE CORRETTO
> auth_source: auto
> ```
