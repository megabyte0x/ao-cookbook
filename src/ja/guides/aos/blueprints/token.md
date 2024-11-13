# トークンブループリント

トークンブループリントは、`ao`でトークンを迅速に構築するための事前設計されたテンプレートです。これは、始めるのに最適な方法であり、ニーズに合わせてカスタマイズできます。

## トークンブループリントの内容

- **バランス**: `Balances`配列は、参加者のトークンバランスを保存するために使用されます。

- **情報ハンドラー**: `info`ハンドラーは、プロセスがトークンのパラメーター（名前、ティッカー、ロゴ、単位など）を取得できるようにします。

- **バランスハンドラー**: `balance`ハンドラーは、プロセスが参加者のトークンバランスを取得できるようにします。

- **バランス一覧ハンドラー**: `balances`ハンドラーは、プロセスがすべての参加者のトークンバランスを取得できるようにします。

- **転送ハンドラー**: `transfer`ハンドラーは、プロセスが他の参加者にトークンを送信できるようにします。

- **ミントハンドラー**: `mint`ハンドラーは、プロセスが新しいトークンをミントできるようにします。

- **総供給ハンドラー**: `totalSupply`ハンドラーは、プロセスがトークンの総供給量を取得できるようにします。

- **バーナーハンドラー**: `burn`ハンドラーは、プロセスがトークンを焼却できるようにします。

### 使用方法:

1. 好みのテキストエディタを開きます。
2. ターミナルを開きます。
3. `aos`プロセスを開始します。
4. `.load-blueprint token`と入力します。

### ブループリントがロードされたか確認する:

`Handlers.list`と入力して、新しくロードされたハンドラーを確認します。

## トークンブループリントの内容:

<!-- # Token Blueprint

The Token Blueprint is a predesigned template that helps you quickly build a token in `ao`. It is a great way to get started and can be customized to fit your needs.

## Unpacking the Token Blueprint

- **Balances**: The `Balances` array is used to store the token balances of the participants.

- **Info Handler**: The `info` handler allows processes to retrieve the token parameters, like Name, Ticker, Logo, and Denomination.

- **Balance Handler**: The `balance` handler allows processes to retrieve the token balance of a participant.

- **Balances Handler**: The `balances` handler allows processes to retrieve the token balances of all participants.

- **Transfer Handler**: The `transfer` handler allows processes to send tokens to another participant.

- **Mint Handler**: The `mint` handler allows processes to mint new tokens.

- **Total Supply Handler**: The `totalSupply` handler allows processes to retrieve the total supply of the token.

- **Burn Handler**: The `burn` handler allows processes to burn tokens.

### How To Use:

1. Open your preferred text editor.
2. Open the Terminal.
3. Start your `aos` process.
4. Type in `.load-blueprint token`

### Verify the Blueprint is Loaded:

Type in `Handlers.list` to see the newly loaded handlers.

## What's in the Token Blueprint: -->

```lua
local bint = require('.bint')(256)
--[[
  This module implements the ao Standard Token Specification.

  Terms:
    Sender: the wallet or Process that sent the Message

  It will first initialize the internal state, and then attach handlers,
    according to the ao Standard Token Spec API:

    - Info(): return the token parameters, like Name, Ticker, Logo, and Denomination

    - Balance(Target?: string): return the token balance of the Target. If Target is not provided, the Sender
        is assumed to be the Target

    - Balances(): return the token balance of all participants

    - Transfer(Target: string, Quantity: number): if the Sender has a sufficient balance, send the specified Quantity
        to the Target. It will also issue a Credit-Notice to the Target and a Debit-Notice to the Sender

    - Mint(Quantity: number): if the Sender matches the Process Owner, then mint the desired Quantity of tokens, adding
        them the Processes' balance
]]
--
local json = require('json')

--[[
  utils helper functions to remove the bint complexity.
]]
--


local utils = {
  add = function(a, b)
    return tostring(bint(a) + bint(b))
  end,
  subtract = function(a, b)
    return tostring(bint(a) - bint(b))
  end,
  toBalanceValue = function(a)
    return tostring(bint(a))
  end,
  toNumber = function(a)
    return bint.tonumber(a)
  end
}


--[[
     Initialize State

     ao.id is equal to the Process.Id
   ]]
--
Variant = "0.0.3"

-- token should be idempotent and not change previous state updates
Denomination = Denomination or 12
Balances = Balances or { [ao.id] = utils.toBalanceValue(10000 * 10 ^ Denomination) }
TotalSupply = TotalSupply or utils.toBalanceValue(10000 * 10 ^ Denomination)
Name = Name or 'Points Coin'
Ticker = Ticker or 'PNTS'
Logo = Logo or 'SBCCXwwecBlDqRLUjb8dYABExTJXLieawf7m2aBJ-KY'

--[[
     Add handlers for each incoming Action defined by the ao Standard Token Specification
   ]]
--

--[[
     Info
   ]]
--
Handlers.add('info', "Info", function(msg)
  msg.reply({
    Name = Name,
    Ticker = Ticker,
    Logo = Logo,
    Denomination = tostring(Denomination)
  })
end)

--[[
     Balance
   ]]
--
Handlers.add('balance', "Balance", function(msg)
  local bal = '0'

  -- If not Recipient is provided, then return the Senders balance
  if (msg.Tags.Recipient) then
    if (Balances[msg.Tags.Recipient]) then
      bal = Balances[msg.Tags.Recipient]
    end
  elseif msg.Tags.Target and Balances[msg.Tags.Target] then
    bal = Balances[msg.Tags.Target]
  elseif Balances[msg.From] then
    bal = Balances[msg.From]
  end

  msg.reply({
    Balance = bal,
    Ticker = Ticker,
    Account = msg.Tags.Recipient or msg.From,
    Data = bal
  })
end)

--[[
     Balances
   ]]
--
Handlers.add('balances', "Balances",
  function(msg) msg.reply({ Data = json.encode(Balances) }) end)

--[[
     Transfer
   ]]
--
Handlers.add('transfer', "Transfer", function(msg)
  assert(type(msg.Recipient) == 'string', 'Recipient is required!')
  assert(type(msg.Quantity) == 'string', 'Quantity is required!')
  assert(bint.__lt(0, bint(msg.Quantity)), 'Quantity must be greater than 0')

  if not Balances[msg.From] then Balances[msg.From] = "0" end
  if not Balances[msg.Recipient] then Balances[msg.Recipient] = "0" end

  if bint(msg.Quantity) <= bint(Balances[msg.From]) then
    Balances[msg.From] = utils.subtract(Balances[msg.From], msg.Quantity)
    Balances[msg.Recipient] = utils.add(Balances[msg.Recipient], msg.Quantity)

    --[[
         Only send the notifications to the Sender and Recipient
         if the Cast tag is not set on the Transfer message
       ]]
    --
    if not msg.Cast then
      -- Debit-Notice message template, that is sent to the Sender of the transfer
      local debitNotice = {
        Action = 'Debit-Notice',
        Recipient = msg.Recipient,
        Quantity = msg.Quantity,
        Data = Colors.gray ..
            "You transferred " ..
            Colors.blue .. msg.Quantity .. Colors.gray .. " to " .. Colors.green .. msg.Recipient .. Colors.reset
      }
      -- Credit-Notice message template, that is sent to the Recipient of the transfer
      local creditNotice = {
        Target = msg.Recipient,
        Action = 'Credit-Notice',
        Sender = msg.From,
        Quantity = msg.Quantity,
        Data = Colors.gray ..
            "You received " ..
            Colors.blue .. msg.Quantity .. Colors.gray .. " from " .. Colors.green .. msg.From .. Colors.reset
      }

      -- Add forwarded tags to the credit and debit notice messages
      for tagName, tagValue in pairs(msg) do
        -- Tags beginning with "X-" are forwarded
        if string.sub(tagName, 1, 2) == "X-" then
          debitNotice[tagName] = tagValue
          creditNotice[tagName] = tagValue
        end
      end

      -- Send Debit-Notice and Credit-Notice
      msg.reply(debitNotice)
      Send(creditNotice)
    end
  else
    msg.reply({
      Action = 'Transfer-Error',
      ['Message-Id'] = msg.Id,
      Error = 'Insufficient Balance!'
    })
  end
end)

--[[
    Mint
   ]]
--
Handlers.add('mint', "Mint", function(msg)
  assert(type(msg.Quantity) == 'string', 'Quantity is required!')
  assert(bint(0) < bint(msg.Quantity), 'Quantity must be greater than zero!')

  if not Balances[ao.id] then Balances[ao.id] = "0" end

  if msg.From == ao.id then
    -- Add tokens to the token pool, according to Quantity
    Balances[msg.From] = utils.add(Balances[msg.From], msg.Quantity)
    TotalSupply = utils.add(TotalSupply, msg.Quantity)
    msg.reply({
      Data = Colors.gray .. "Successfully minted " .. Colors.blue .. msg.Quantity .. Colors.reset
    })
  else
    msg.reply({
      Action = 'Mint-Error',
      ['Message-Id'] = msg.Id,
      Error = 'Only the Process Id can mint new ' .. Ticker .. ' tokens!'
    })
  end
end)

--[[
     Total Supply
   ]]
--
Handlers.add('totalSupply', "Total-Supply", function(msg)
  assert(msg.From ~= ao.id, 'Cannot call Total-Supply from the same process!')

  msg.reply({
    Action = 'Total-Supply',
    Data = TotalSupply,
    Ticker = Ticker
  })
end)

--[[
 Burn
]] --
Handlers.add('burn', 'Burn', function(msg)
  assert(type(msg.Quantity) == 'string', 'Quantity is required!')
  assert(bint(msg.Quantity) <= bint(Balances[msg.From]), 'Quantity must be less than or equal to the current balance!')

  Balances[msg.From] = utils.subtract(Balances[msg.From], msg.Quantity)
  TotalSupply = utils.subtract(TotalSupply, msg.Quantity)

  msg.reply({
    Data = Colors.gray .. "Successfully burned " .. Colors.blue .. msg.Quantity .. Colors.reset
  })
end)
```