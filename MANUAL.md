# qbx_towjob — Manual

Emprego de guincho: o jogador retira um flatbed mediante depósito, reboca veículos espalhados pelo mapa e recebe o pagamento no fim do turno.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Comandos](#comandos)
5. [Fluxo do trabalho](#fluxo-do-trabalho)
6. [Depósito e pagamento](#depósito-e-pagamento)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | Framework base: `GetPlayer`, `Notify`, `qbx.spawnVehicle`, dados de job |
| `ox_lib` | Sim | Locale, zonas, `progressBar`, `lib.addCommand`, callbacks |
| `ox_target` | Não | Usado apenas no ponto de pagamento quando a convar `UseTarget` está ativa |
| `qbx_vehiclekeys` | Não | Recebe `vehiclekeys:client:SetOwner` ao spawnar o flatbed |

O recurso declara `provide 'qb-towjob'`, então substitui o `qb-towjob` legado.

---

## Instalação

1. Copie a pasta `qbx_towjob` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_towjob
   ```
3. Certifique-se de que existe o job `tow` no `qbx_core` (`shared/jobs.lua`). O comando `/tow` também aceita o job `mechanic`.
4. Opcional — para usar o `ox_target` no ponto de pagamento:
   ```
   setr UseTarget true
   ```
5. **Conflitos** — não rode junto com o `qb-towjob`; os dois registram os mesmos eventos `qb-tow:*`.

---

## Configuração

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `debugPoly` | bool | Não | Desenha as zonas (HQ, garagem, pontos de reboque) para depuração |
| `useTarget` | bool | Não | Lido da convar `UseTarget`. Quando `true`, o ponto de pagamento vira uma opção de `ox_target` em vez de zona automática |
| `vehicles` | tabela `[spawnName] = label` | Sim | Veículos disponíveis no menu da garagem. Padrão: `flatbed`. Usada também para validar se o jogador está num veículo de reboque |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `bailPrice` | number | Sim | Depósito cobrado ao retirar o flatbed, devolvido ao guardá-lo. Padrão: `250` |
| `paymentTax` | number | Sim | Percentual de imposto descontado do pagamento final. Padrão: `15` |

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `locations.main` | `{ label, coords: vec4 }` | Sim | Sede do guincho. Zona (ou opção de target) onde se recebe o pagamento do turno |
| `locations.vehicle` | `{ label, coords: vec4 }` | Sim | Garagem: retirada e devolução do flatbed |
| `locations.dropoff` | `{ label, coords: vec3 }` | Sim | Depósito onde o veículo rebocado deve ser entregue |
| `locations.towspots` | array de `{ model, coords: vec3 }` | Sim | Pontos onde os veículos do NPC podem aparecer. O recurso sorteia um por vez, nunca repetindo o anterior imediatamente. Padrão: 40 pontos |

Cada entrada de `towspots` define o **modelo exato** do veículo que será spawnado ali. Ao rebocar, o recurso valida se o veículo alvo é desse modelo.

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/npc` | Job `tow` | Liga/desliga o modo de missões NPC. Ao ligar, sorteia um ponto de reboque e traça a rota. Não pode ser desligado com um veículo ainda rebocado |
| `/tow` | Job `tow` ou `mechanic` | Engata o veículo atrás do flatbed, ou desengata se já houver um veículo rebocado. Precisa estar dentro do flatbed |

---

## Fluxo do trabalho

1. Com o job `tow`, os blips da sede e da garagem aparecem no mapa.
2. Na garagem (`locations.vehicle`), o jogador é levado ao menu de veículos e paga o depósito para tirar o flatbed. A placa é gerada como `TOWR` + 4 dígitos.
3. `/npc` inicia o modo de missões: um ponto de reboque é sorteado e recebe blip com rota.
4. Ao chegar na zona (50x50), o veículo alvo é spawnado com o tanque vazio.
5. Dentro do flatbed, `/tow` engata o veículo alvo (barra de progresso de 5 s, alcance máximo de 11 m). Só o modelo correto é aceito enquanto o modo NPC está ligado.
6. Um blip aponta para o depósito (`locations.dropoff`), com marcador no chão. Chegando lá, `/tow` desengata: se estiver a menos de 25 m da garagem, o veículo é deletado, o contador de entregas sobe e um novo ponto é sorteado.
7. Na sede (`locations.main`), a interação de pagamento fecha o turno e paga todas as entregas acumuladas.
8. Para guardar o flatbed, volte à garagem já sem veículo rebocado: o veículo é deletado e o depósito devolvido.

---

## Depósito e pagamento

**Depósito** — o valor de `bailPrice` é cobrado em dinheiro se houver saldo em espécie suficiente; caso contrário, do banco. O servidor exige que o jogador tenha **pelo menos `bailPrice` em dinheiro E no banco** para liberar o flatbed. A devolução é sempre creditada no **banco**.

**Pagamento** — calculado no servidor ao interagir com a sede:

- Cada entrega vale um valor aleatório entre `150` e `170` (mesmo valor para todas as entregas do turno).
- Acima de 5 entregas, entra um bônus progressivo. O total de entregas considerado é limitado a `20`.
- Sobre o total incide `paymentTax`% de imposto.
- O líquido é creditado no banco e o jogador ganha 1 ponto de reputação de job (`AddJobReputation`).

O servidor valida que o jogador tem o job `tow` e está a menos de 6 m da sede; caso contrário, o jogador é desconectado por tentativa de exploit.

---

## Entrypoints para outros recursos

Não há exports. Os eventos abaixo são os pontos de entrada reais:

```lua
-- Liga/desliga o modo de missões NPC do jogador (equivalente a /npc)
TriggerClientEvent('jobs:client:ToggleNpc', source)

-- Executa a ação de engatar/desengatar (equivalente a /tow)
TriggerClientEvent('qb-tow:client:TowVehicle', source)
```

Callback registrado no servidor (usado internamente pelo cliente):

```lua
lib.callback.await('qb-tow:server:spawnVehicle', false, model, coords, warp)
```

---

## Localização

As strings vêm do sistema de locale do `ox_lib`. Idiomas em `locales/`: `bg`, `cs`, `de`, `en`, `es`, `fr`, `pl`, `pt`, `pt-br`, `ro`, `tr`.

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_towjob/
├── client/
│   └── main.lua          — zonas, blips, menu da garagem, engate/desengate, marcadores
├── server/
│   └── main.lua          — depósito, cálculo do pagamento, comandos /npc e /tow, spawn de veículos
├── config/
│   ├── client.lua        — debugPoly, useTarget, lista de veículos de reboque
│   ├── server.lua        — bailPrice, paymentTax
│   └── shared.lua        — sede, garagem, depósito e os 40 pontos de reboque
├── locales/              — traduções (bg, cs, de, en, es, fr, pl, pt, pt-br, ro, tr)
└── fxmanifest.lua
```
