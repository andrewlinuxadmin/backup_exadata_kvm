# backup_exadata_kvm

Backup de VMs KVM de um appliance Exadata para um volume NFS, usando reflink + tar
(procedimento oficial Oracle).

- **Role `backup_exadata_kvm`**: faz o backup de **uma** VM.
- **Playbook `backup_kvm.yml`**: valida o NFS, localiza as VMs, executa a role em loop
  e aplica a retenção.

## Requisitos

- ansible-core 2.14 ou superior (somente módulos `ansible.builtin`).
- Acesso root (`become`) ao host KVM do Exadata.
- Volume NFS montado no destino (padrão `/mnt`).
- Diretório temporário no mesmo filesystem das imagens das VMs (necessário para reflink).

## Uso

```bash
ansible-playbook -i inventario backup_kvm.yml
```

Exemplos:

```bash
# Limitar a um host do inventário e usar outro padrão de VMs
ansible-playbook -i inventario backup_kvm.yml \
  -l exadata01 \
  -e 'vm_pattern=vm*.dominio.local'

# Interromper na primeira falha (comportamento do script original)
ansible-playbook -i inventario backup_kvm.yml -e backup_exadata_kvm_fail_on_error=true
```

## Fluxo

**Playbook**

1. Verifica se `backup_exadata_kvm_dest_base` está montado via NFS.
2. Gera um timestamp único para a execução.
3. Localiza as VMs em `backup_exadata_kvm_guest_images_dir` pelo padrão `vm_pattern`.
4. Verifica o espaço no NFS, calcula a retenção e remove backups antigos (ver [Espaço e retenção](#espaço-e-retenção)).
5. Executa a role para cada VM, sequencialmente.
6. Falha ao final se algum backup falhou, listando as VMs.

**Role (por VM)**

1. Cria o diretório temporário `<tmp_base>/Backup-<vm>-<timestamp>`.
2. Copia o XML do libvirt e cria reflinks das imagens da VM (ignora `lost+found`).
3. Gera o tar no NFS (`--acls --xattrs --format=pax`).
4. Valida o retorno do tar e compara o arquivo com a origem (`tar --compare`).
5. Remove o diretório temporário (também removido em caso de falha).
6. Grava checksum e relatório do backup.

## Estrutura no NFS

```
<dest_base>/<timestamp>/<vm>/
├── <vm>-<timestamp>.tar
├── <vm>-<timestamp>.tar.sha256   # verificável com: sha256sum -c
├── tar.stdout.log                # lista de arquivos do tar
├── tar.stderr.log                # erros do tar
├── tar.verify.log                # resultado do tar --compare
├── backup_report.yml             # gravado somente em caso de sucesso
└── backup_failed.yml             # gravado somente em caso de falha
```

O `backup_report.yml` contém VM, host, início/fim, tamanho, checksum, códigos de retorno
do tar e da verificação, e a lista de arquivos incluídos.

## Espaço e retenção

Executado pelo playbook **antes** do backup:

1. Obtém o tamanho total e o espaço livre do volume NFS.
2. Calcula o espaço necessário: soma do tamanho das VMs + `space_margin_pct`.
   É usado o tamanho aparente dos arquivos, pois o tar grava imagens esparsas por inteiro.
3. Falha se o backup for maior que o volume NFS inteiro.
4. Calcula quantos backups cabem no volume (`total / necessário`). Essa é a retenção,
   limitada por `retention` quando maior que 0.
5. Planeja a remoção, do backup mais antigo para o mais novo:
   - os que excedem a retenção (contando o novo backup), e
   - apenas os necessários para que o espaço livre comporte o novo backup.
6. Falha, sem remover nada, se mesmo removendo todos os backups antigos não houver espaço.
7. Remove os backups planejados.

Regras:

- Considera apenas pastas `<timestamp>` que tenham ao menos um `backup_report.yml`,
  para não apagar outros conteúdos do NFS.
- Remove a pasta `<timestamp>` inteira (todas as VMs daquela execução).
- Outros conteúdos do NFS ocupam espaço mas não são removidos; se impedirem o backup, o playbook falha.

## Variáveis

### Playbook

| Variável | Padrão | Descrição |
|---|---|---|
| `vm_pattern` | `exa04*.intra.local` | Padrão (glob) do nome das VMs |
| `retention` | `0` | Backups mantidos no NFS. `0` = automático (quantos cabem no volume); `> 0` = limite máximo |
| `space_margin_pct` | `10` | Margem (%) sobre o tamanho das VMs para estimar o backup |
| `backup_exadata_kvm_fail_on_error` | `false` | `true` interrompe na primeira VM com falha |

### Role

| Variável | Padrão | Descrição |
|---|---|---|
| `backup_exadata_kvm_vm_name` | — | Nome da VM (obrigatório) |
| `backup_exadata_kvm_guest_images_dir` | `/EXAVMIMAGES/GuestImages` | Diretório das imagens das VMs |
| `backup_exadata_kvm_libvirt_xml_dir` | `/etc/libvirt/qemu` | Diretório dos XMLs do libvirt |
| `backup_exadata_kvm_tmp_base` | `/EXAVMIMAGES/backups` | Base do diretório temporário (mesmo filesystem das imagens) |
| `backup_exadata_kvm_dest_base` | `/mnt` | Ponto de montagem NFS de destino |
| `backup_exadata_kvm_timestamp` | gerado | Timestamp do backup |
| `backup_exadata_kvm_timestamp_format` | `%Y-%m-%d-%Hh%M` | Formato do timestamp |
| `backup_exadata_kvm_reflink` | `always` | Modo do `cp --reflink` (`always` falha sem suporte a reflink) |
| `backup_exadata_kvm_tar_options` | `--acls --xattrs --xattrs-include=* --format=pax` | Opções do tar |
| `backup_exadata_kvm_verify` | `true` | Executa `tar --compare` após o tar |
| `backup_exadata_kvm_checksum` | `true` | Gera o arquivo de checksum |
| `backup_exadata_kvm_checksum_algorithm` | `sha256` | Algoritmo do checksum |
| `backup_exadata_kvm_report_file` | `backup_report.yml` | Nome do relatório de sucesso |
| `backup_exadata_kvm_failed_file` | `backup_failed.yml` | Nome do relatório de falha |
| `backup_exadata_kvm_fail_on_error` | `true` | `false` registra a falha e segue sem erro |
| `backup_exadata_kvm_async_timeout` | `86400` | Tempo máximo (s) do tar, verificação e checksum |
| `backup_exadata_kvm_async_poll` | `60` | Intervalo (s) de acompanhamento das tarefas longas |

> `backup_exadata_kvm_dest_base` e `backup_exadata_kvm_guest_images_dir` são definidas no
> playbook e repassadas automaticamente à role.

## Observações

- `tar --compare` e o checksum leem o arquivo inteiro novamente pelo NFS, o que pode mais
  que dobrar o tempo de backup. Desative com `backup_exadata_kvm_verify=false` e/ou
  `backup_exadata_kvm_checksum=false` se necessário.
- São usadas três tarefas `command` (sem shell), por não haver módulo equivalente:
  `cp --reflink`, `tar` (o módulo `archive` não preserva ACLs/xattrs nem gera pax)
  e `tar --compare`.
- O timestamp é gerado com o relógio do controlador Ansible.

## Restore (referência)

> Extrair com `-C /` sobrescreve as imagens em `/EXAVMIMAGES/GuestImages/<vm>`.
> Faça com a VM parada e siga o procedimento de restore da documentação Oracle.

```bash
cd <dest_base>/<timestamp>/<vm>/
sha256sum -c <vm>-<timestamp>.tar.sha256
tar --acls --xattrs --xattrs-include='*' -xpvf <vm>-<timestamp>.tar -C /
# XML do libvirt extraído em /XML/<vm>.xml
```
