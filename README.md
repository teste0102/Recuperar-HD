# Recuperar-HD

Roteiro e anotações para diagnosticar e tentar recuperar HDs no Windows com
**openSeaChest** (Seagate) e **Victoria**. Rodar no PowerShell como Administrador,
dentro de `C:\IA_@\openseachest\` (no PowerShell, programas locais levam `.\` na frente).

> ⚠️ Conferir sempre modelo e serial no `--scan` antes de qualquer comando.
> O número do disco (`PD1`, `PD2`...) pode mudar. Nunca mexer no SSD.

## Passo a passo

```powershell
# 1. achar o disco
.\openSeaChest_PowerControl.exe --scan

# 2. ver os atributos SMART (raw | hybrid | analyzed)
.\openSeaChest_SMART.exe -d PD1 --smartAttributes raw

# 3. APM: parar de estacionar a cabeça (pode voltar a 128 após religar o disco)
.\openSeaChest_PowerControl.exe -d PD1 --showAPMLevel
.\openSeaChest_PowerControl.exe -d PD1 --setAPMLevel 254

# 4. leitura completa que reescreve (remapeia) só os setores ilegíveis, 3-5 h
.\openSeaChest_GenericTests.exe -d PD1 --longGeneric --repairAtEnd --errorLimit 5000

# 5. conferir de novo
.\openSeaChest_SMART.exe -d PD1 --smartAttributes raw
```

Notas:
- `--showEPCSettings` não é suportado em discos antigos (ex.: ST500LT012).
- A flag do APM fica no **PowerControl** (`--setAPMLevel`), não no Configure.
- O `--errorLimit` é em número de LBAs. Sem ele o teste aborta com ~400 erros seguidos.
- `--repairAtEnd` só reescreve setores ruins; o resto do disco não é tocado.
- Desativar "desligar disco rígido" nas Opções de Energia e a desfragmentação agendada.

## Atributos SMART que importam

| ID | Significado | Sinal ruim |
|---|---|---|
| 5 | Setores realocados | Valor perto do limite (thresh) = reserva esgotada |
| 187 | Erros não corrigíveis | Valor 1 / contador saturado |
| 197 | Pendentes | > 0 (o reparo tenta corrigir) |
| 198 | Offline não corrigíveis | > 0 |
| 199 | CRC UDMA | Sobe = cabo ou conector ruim |

## Histórico de discos testados

### ST500LT012-9WS142 — serial W0VB30JD — **descartado**
- ID 5 = 16384 (valor 01, limite 36): reserva de setores esgotada.
- ID 187 = 11699; 193 = 95759; choques (191) = 1552.
- Leitura completa abortou com **401 LBAs seguidos ilegíveis** (83714944–83715344).
- Depois do teste o disco parou de responder (sem modelo/serial). Não serve para Xbox, NAS ou Frigate.

### ST9500325AS — serial S2WR9XGH — **em teste**
- ID 5 = 218 (valor 90, limite 36): reserva boa.
- ID 197 = 2, 198 = 2, 199 = 6; ID 187 saturado (raw 65535, valor 1). Status Victoria: Caution.
- APM mudou de 128 para 254 (conferir após religar).
- Próximo: rodar o passo 4 e comparar 5, 197 e 198.
