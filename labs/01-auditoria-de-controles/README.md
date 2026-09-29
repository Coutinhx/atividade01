# Atividade 01 – Planejamento e execução da auditoria: auditoria de controles

> **Professor Esp. Marcelino Dias da Silva Junior** · UNINOVE · Disciplina: **Auditoria Forense de Sistemas Digitais**
> **Aula relacionada:** Planejamento e execução da auditoria (reforça: Introdução à auditoria, Relatórios preliminares)

## 🎯 Objetivo
Executar uma auditoria simples: comparar a configuração de um servidor com **critérios** definidos no planejamento e registrar cada **evidência** num papel de trabalho.

## 📚 Conceito em 1 minuto
- **Planejamento:** antes de começar, a auditoria define **objetivo, escopo e critérios**.
- **Execução:** o auditor coleta **evidências** que mostram se cada critério foi cumprido.
- **Não conformidade (NC):** critério não atendido, descrita de forma **clara, firme e objetiva** (fato, critério e efeito).

**Escopo desta auditoria:** servidor **FIN-SRV01** (setor financeiro), usando uma **cópia** das configurações.

| Critério | Regra (política da empresa) |
|---|---|
| C1 | Somente a conta `root` pode ter UID 0 (privilégio máximo) |
| C2 | Nenhuma conta pode ficar sem senha |
| C3 | Arquivos do financeiro não podem ter permissão de escrita para "outros" |
| C4 | Contas de funcionários desligados devem ser removidas |

## 🧪 Passo a passo

```bash
cd /workspaces/*/labs/01-auditoria-de-controles
bash preparar.sh
tree saida/servidor
```

**C1: contas com UID 0.** O 3º campo do `passwd` é o UID:

```bash
awk -F: '$3 == 0 {print $1, "-> UID", $3}' saida/servidor/etc/passwd
```

**C2: contas sem senha.** No `shadow`, o 2º campo vazio significa conta sem senha:

```bash
awk -F: '$2 == "" {print "SEM SENHA:", $1}' saida/servidor/etc/shadow
```

**C3: arquivos com escrita para "outros".**

```bash
ls -l saida/servidor/financeiro/
find saida/servidor/financeiro -type f -perm -o+w
```

**C4: contas de funcionários desligados.** Leia a descrição (5º campo):

```bash
cut -d: -f1,5 saida/servidor/etc/passwd
grep -i "desligad" saida/servidor/etc/passwd
```

**Guarde as evidências com hash**, para que ninguém possa dizer que foram alteradas depois:

```bash
{ echo "Auditoria FIN-SRV01 - $(date -u '+%Y-%m-%d %H:%M UTC') - auditor: $(whoami)"
  awk -F: '$3 == 0' saida/servidor/etc/passwd
  awk -F: '$2 == ""' saida/servidor/etc/shadow
  find saida/servidor/financeiro -type f -perm -o+w
  grep -i "desligad" saida/servidor/etc/passwd
} > saida/evidencias_auditoria.txt
sha256sum saida/evidencias_auditoria.txt | tee saida/evidencias_auditoria.sha256
```

## 📝 Papel de trabalho (preencha)

| Critério | Evidência (comando + resultado) | Conforme? | NC |
|---|---|---|---|
| C1 | `awk -F: '$3 == 0 {print $1, "-> UID", $3}' ...` → `root -> UID 0`; `suporte -> UID 0` | Não | NC-01: conta `suporte` também tem UID 0 |
| C2 | `awk -F: '$2 == "" {print "SEM SENHA:", $1}' ...` → `SEM SENHA: suporte` | Não | NC-02: senha vazia na conta `suporte` |
| C3 | `find ... -type f -perm -o+w` → `saida/servidor/financeiro/pagamentos.csv`; `ls -l` mostrou `-rwxrwxrwx` | Não | NC-03: escrita permitida a outros em `pagamentos.csv` |
| C4 | `grep -i "desligad" ...` → `estagiario2024` (desligado em 2024) | Não | NC-04: conta de ex-funcionário ainda existe |

## ❓ Perguntas
1. A conta `suporte` viola C1 e C2: tem UID 0, como `root`, e está sem senha. Isso permite que alguém que obtenha acesso à conta alcance privilégios máximos sem autenticação adequada, comprometendo os dados e o servidor financeiro.
2. **NC-01 — Fato:** a conta `suporte` está configurada com UID 0. **Critério:** pela política C1, somente `root` pode ter UID 0. **Efeito:** a conta de suporte também recebe privilégio máximo, ampliando o risco de acesso e alterações não autorizadas no servidor.
3. A análise foi feita numa cópia para não alterar configurações nem interromper o servidor em produção. A cópia mantém a atividade controlada e permite repetir os testes sem comprometer a operação ou a integridade das evidências originais.
4. **NC-01:** remover o UID 0 de `suporte` e conceder apenas os privilégios necessários. **NC-02:** definir uma senha forte e individual ou desabilitar a conta até regularizá-la. **NC-03:** retirar a permissão de escrita para "outros" em `pagamentos.csv` e revisar proprietário, grupo e permissões dos arquivos financeiros. **NC-04:** desabilitar e remover a conta `estagiario2024`, revogando seus acessos conforme o processo de desligamento.
