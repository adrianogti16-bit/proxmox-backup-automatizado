# Backup de VMs/Containers Proxmox VE com Proxmox Backup Server (PBS) + Sincronização com Google Drive

## 📋 Visão geral

Este documento descreve o procedimento completo para:

1. Integrar um **Proxmox VE** (host de virtualização) a um **Proxmox Backup Server (PBS)**
2. Criar e agendar um job de backup automático para VMs e Containers
3. Validar a execução do backup e interpretar avisos (warnings) do log
4. Configurar o **rclone** no PBS para sincronizar os backups com o **Google Drive**, como camada extra de proteção (3-2-1 backup)

**Ambiente utilizado:**
- Proxmox VE 8.x
- Proxmox Backup Server 4.2.0
- Windows 10/11 (máquina cliente para autenticação OAuth)

---

## 1. Conectando o Proxmox VE ao PBS

No **Proxmox VE**, vá em **Datacenter → Storage → Add → Proxmox Backup Server** e preencha:

| Campo       | Valor de exemplo         |
|-------------|---------------------------|
| ID          | `PBS`                      |
| Server      | `IP_DO_SEU_PBS`             |
| Username    | `root@pam`                  |
| Password    | *(senha do usuário no PBS)* |
| Datastore   | `backup-principal`          |

### ⚠️ Erro comum: fingerprint não verificado

Ao clicar em "Add", pode aparecer o erro:

```
create storage failed: PBS: error fetching datastores - fingerprint 'XX:XX:...' not verified, abort! (500)
```

**Causa:** o certificado TLS do PBS é autoassinado e o Proxmox VE não confia nele automaticamente.

**Solução:** obter o fingerprint SHA-256 direto no Shell do PBS:

```bash
openssl x509 -noout -fingerprint -sha256 -in /etc/proxmox-backup/proxy.pem
```

Copiar o valor exibido (após o `=`) e colar no campo **Fingerprint** da tela de adição do storage no Proxmox VE. Em seguida, clicar em "Add" novamente.

---

## 2. Criando o job de backup agendado

No **Proxmox VE**, ir em **Datacenter → Backup → Add** e configurar:

| Campo           | Valor                          |
|-----------------|---------------------------------|
| Node            | `-- All --`                     |
| Storage         | Storage do PBS (ex: `PBS`)      |
| Schedule        | `02:00` (ou horário desejado)   |
| Selection mode  | `Include selected VMs`          |
| Mode            | `Snapshot`                      |
| Compression     | `ZSTD (fast and good)`          |

Selecionar manualmente as VMs/Containers que devem ser incluídos no backup. 

> 💡 **Dica:** se o próprio PBS estiver rodando como uma VM dentro do mesmo cluster, **não** o inclua no job de backup dele mesmo — isso não protege contra falha do host onde ele está.

---

## 3. Executando e validando o backup manualmente

Para testar sem esperar o agendamento, selecionar o job criado em **Datacenter → Backup** e clicar em **"Run now"**.

Acompanhar o progresso na janela de log (aba **Output**). Ao final, checar a aba **Status**:

- `Status: OK` → backup concluído sem problemas
- `Status: stopped: WARNINGS: N` → concluído com sucesso, mas com avisos a revisar

### Avisos comuns encontrados

```
WARN: iothread is not valid with virtio disk or virtio-scsi-single controller, ignoring
WARN: swtpm_setup: Not overwriting existing state file.
```

- O primeiro é um aviso de **configuração de disco** da VM (opção "IO thread" incompatível com o controlador usado) — não impede o backup, apenas é ignorado.
- O segundo é relacionado ao **TPM virtual** (usado por VMs Windows) — indica que o estado do TPM já existia e não foi sobrescrito, comportamento normal.

Ambos **não indicam falha** no backup.

### Verificando os backups no PBS

Acessar a interface web do PBS → **Datastore → [nome do datastore] → Content**, onde é possível ver todos os backups (`vm/ID` ou `ct/ID`) com data, hora e tamanho.

---

## 4. Sincronizando os backups com o Google Drive (rclone)

Como camada extra de proteção (regra 3-2-1: 3 cópias, 2 mídias diferentes, 1 fora do local), os backups do datastore local do PBS podem ser sincronizados para um Google Drive usando **rclone**.

### 4.1 Instalar o rclone no PBS

No Shell do PBS:

```bash
apt update
apt install rclone -y
```

### 4.2 Configurar o acesso ao Google Drive

```bash
rclone config
```

Sequência de respostas:

1. `n` → novo remote
2. Nome do remote (ex: `gdrive`)
3. Tipo: selecionar o número correspondente a **Google Drive**
4. `client_id`: deixar em branco (Enter)
5. `client_secret`: deixar em branco (Enter)
6. `scope`: `1` (Full access all files)
7. `service_account_file`: deixar em branco (Enter)
8. `Edit advanced config?`: `n`
9. `Use auto config?`: **`n`** (obrigatório, pois o PBS é headless/sem navegador)

### 4.3 Autorizar via máquina com navegador

O terminal do PBS vai exibir um comando parecido com:

```bash
rclone authorize "drive" "TOKEN_DE_SCOPE"
```

Executar esse **mesmo comando** em um computador com navegador (Windows, no caso, usando `rclone.exe`), baixado de: https://rclone.org/downloads/

```powershell
rclone.exe authorize "drive" "TOKEN_DE_SCOPE"
```

Isso abre o navegador para login e autorização na conta Google. Ao final, aparece a mensagem **"Success!"** e o terminal do Windows exibe um bloco de texto (JSON) entre as linhas:

```
Paste the following into your remote machine --->
{...token json...}
<---End paste
```

Copiar **exatamente esse bloco** e colar de volta no terminal do PBS, no prompt `config_token>`.

> ⚠️ **Atenção:** essa etapa deve ser feita **rapidamente**, sem interromper a sessão (evitar `Ctrl+C`), pois o processo pode expirar ou ser cancelado, sobrando remotes "quebrados" na configuração (erro `empty token found`).

### 4.4 Perguntas finais

- `Configure this as a Shared Drive (Team Drive)?` → `n` (a menos que use Google Workspace com Drive Compartilhado)
- Confirmar a configuração: `y`

### 4.5 Testar a conexão

```bash
rclone lsd gdrive:
```

Se listar as pastas reais do Google Drive, a configuração está correta.

### 4.6 Limpando remotes duplicados/quebrados (se houver)

```bash
rclone config delete NOME_DO_REMOTE_QUEBRADO
```

Para renomear um remote:

```bash
rclone config
# escolher: r (Rename remote)
# selecionar o número do remote atual
# digitar o novo nome
```

---

## 5. Próximos passos (a implementar)

- [ ] Criar rotina de sincronização automática (`rclone sync`) do datastore do PBS para o Google Drive, via `cron`
- [ ] Definir política de retenção (Prune) tanto no PBS quanto no Google Drive
- [ ] Testar restauração de um backup a partir do PBS
- [ ] Monitorar espaço utilizado no datastore e na conta do Google Drive

---

## 📝 Notas de segurança

- IPs, senhas e tokens reais **não foram incluídos** neste documento — substitua os placeholders pelos valores do seu ambiente ao aplicar este procedimento.
- Recomenda-se usar uma conta de serviço dedicada (não a pessoal) para automações de longo prazo com o Google Drive.
