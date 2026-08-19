# Changements du programme MABA par rapport a `maba-default`

Ce document resume les differences reelles observees entre :

```bash
/mnt/c/Users/User/Desktop/wsl_solana/maba-default/programs
```

et :

```bash
/mnt/c/Users/User/Desktop/wsl_solana/mabalayerzero/maba/programs
```

Commande utilisee pour comparer :

```bash
diff -ru /mnt/c/Users/User/Desktop/wsl_solana/maba-default/programs \
  /mnt/c/Users/User/Desktop/wsl_solana/mabalayerzero/maba/programs
```

## Resume

Les changements utiles au bridge MABA sont limites a trois fichiers du programme OFT :

```text
programs/oft/src/instructions/lz_receive.rs
programs/oft/src/instructions/lz_receive_types.rs
programs/oft/src/lib.rs
```

Un quatrieme diff existe sur le mock endpoint, mais il ne concerne pas l'OFT deploye :

```text
programs/endpoint-mock/src/lib.rs
```

## 1. Support de `LzReceiveTypesV2`

Fichiers concernes :

```text
programs/oft/src/instructions/lz_receive_types.rs
programs/oft/src/lib.rs
```

### Probleme initial

LayerZero Executor echouait avant meme d'appeler correctement `lz_receive` avec :

```text
Failed to get lz_receive_types_info, receiver program is not using LzReceiveTypesV2
```

Cela voulait dire que l'executor Solana ne trouvait pas les entrypoints V2 attendus pour recuperer les comptes necessaires a l'execution du message.

### Changement ajoute

Dans `lz_receive_types.rs`, ajout des imports V2 :

```rust
use oapp::{
    common::{AccountMetaRef, AddressLocator, EXECUTION_CONTEXT_VERSION_1},
    endpoint_cpi::LzAccount,
    lz_receive_types_v2::{
        Instruction, LzReceiveTypesV2Accounts, LzReceiveTypesV2Result,
        LZ_RECEIVE_TYPES_VERSION,
    },
};
```

Ajout du compte `LzReceiveTypesInfo` :

```rust
#[derive(Accounts)]
pub struct LzReceiveTypesInfo<'info> {
    pub oapp_account: Account<'info, OFTStore>,
    #[account(
        seeds = [LZ_RECEIVE_TYPES_SEED, oapp_account.key().as_ref()],
        bump,
        constraint = lz_receive_types_accounts.oft_store == oapp_account.key()
    )]
    pub lz_receive_types_accounts: Account<'info, LzReceiveTypesAccounts>,
}
```

Ajout de la fonction demandee :

```rust
impl LzReceiveTypesInfo<'_> {
    pub fn apply(ctx: &Context<LzReceiveTypesInfo>) -> Result<(u8, LzReceiveTypesV2Accounts)> {
        Ok((
            LZ_RECEIVE_TYPES_VERSION,
            LzReceiveTypesV2Accounts {
                accounts: vec![
                    ctx.accounts.oapp_account.key(),
                    ctx.accounts.lz_receive_types_accounts.token_mint,
                ],
            },
        ))
    }
}
```

Ajout de `apply_v2`, qui reutilise la logique V1 existante pour produire le format V2 :

```rust
pub fn apply_v2(
    ctx: &Context<LzReceiveTypes>,
    params: &LzReceiveParams,
) -> Result<LzReceiveTypesV2Result> {
    let accounts = Self::apply(ctx, params)?
        .into_iter()
        .map(|account| AccountMetaRef {
            pubkey: if account.is_signer && account.pubkey == Pubkey::default() {
                AddressLocator::Payer
            } else {
                account.pubkey.into()
            },
            is_writable: account.is_writable,
        })
        .collect();

    Ok(LzReceiveTypesV2Result {
        context_version: EXECUTION_CONTEXT_VERSION_1,
        alts: vec![],
        instructions: vec![Instruction::LzReceive { accounts }],
    })
}
```

Dans `lib.rs`, exposition des deux handlers Anchor :

```rust
pub fn lz_receive_types_info(
    ctx: Context<LzReceiveTypesInfo>,
) -> Result<(u8, oapp::lz_receive_types_v2::LzReceiveTypesV2Accounts)> {
    LzReceiveTypesInfo::apply(&ctx)
}

pub fn lz_receive_types_v2(
    ctx: Context<LzReceiveTypes>,
    params: LzReceiveParams,
) -> Result<oapp::lz_receive_types_v2::LzReceiveTypesV2Result> {
    LzReceiveTypes::apply_v2(&ctx, &params)
}
```

### Effet

LayerZero Executor peut maintenant recuperer les comptes dynamiques en V2 et passer a l'execution de `lz_receive`.

## 2. Ne pas mint si le compte destination est frozen

Fichier concerne :

```text
programs/oft/src/instructions/lz_receive.rs
```

### Objectif

Quand un message Sepolia -> Solana arrive vers une ATA Token-2022 frozen, le programme ne doit pas revert. Il doit consommer le message LayerZero proprement, mais ne pas mint les tokens.

### Changement ajoute

Dans la branche native mint, avant `mint_to` :

```rust
if ctx.accounts.token_dest.is_frozen() {
    emit_cpi!(OFTReceived {
        guid: params.guid,
        src_eid: params.src_eid,
        to: ctx.accounts.to_address.key(),
        amount_received_ld: 0,
    });
    return Ok(());
}
```

### Effet

Comportement apres changement :

```text
ATA non frozen -> mint normal
ATA frozen     -> pas de mint, return Ok(())
```

Le message LayerZero est donc marque comme livre, sans bloquer la pathway.

## 3. Build avec le bon `OFT_ID`

Le programme utilise un `declare_id` derive de l'environnement :

```rust
declare_id!(program_id_from_env!("OFT_ID", "..."));
```

Il faut donc build avec le bon program id deploye :

```bash
OFT_ID=65LAy9QG62cNAYGVMxVEksXw2wPKsaBgM2JrHs1tHctG anchor build
```

Sans cela, la simulation Anchor retournait :

```text
DeclaredProgramIdMismatch
```

## 4. Changement hors OFT : `endpoint-mock`

Fichier concerne :

```text
programs/endpoint-mock/src/lib.rs
```

Diff observe :

```diff
-declare_id!("6xmPjYnXyxz36xcKkv2zCrZc72LK5hQ9xzY3EjeZ59MV");
+declare_id!("CB7sjoqCTThCR8yHE7YcMPXZuBcyxnDYv5ocSGMzExJQ");
```

Ce changement concerne le mock endpoint local, pas le programme OFT MABA deploye.

## Validation effectuee

### Envoi vers une ATA non frozen

Destination wallet :

```text
8AKrTEJSysDdi4ZBAdimkQe6d2wrBVF9K9vLPuBwKQqy
```

Resultat :

```text
LayerZero status: DELIVERED
ATA existe: true
ATA frozen: false
amountRaw: 100
amountUi: 1
supplyRaw: 100
```

### Envoi vers une ATA frozen

ATA :

```text
D2WoVtxsjmuB8vtHyi1dR3tXfDfCAyeApzTRn3UjFdoB
```

Owner wallet :

```text
6qYCnUJvei3BrSwksVRpYs67QeWrK9HeEtF8wEGYZmqJ
```

Resultat :

```text
LayerZero status: DELIVERED
ATA frozen: true
amountRaw: 5
amountUi: 0.05
supplyRaw: 100
```

La supply n'a pas augmente, ce qui confirme que le programme a bien consomme le message sans mint.

## Note LayerZero options

Pour que l'executor Solana passe jusqu'au `PostExecute`, la valeur native de l'option Solana a ete augmentee dans `layerzero.config.ts` :

```ts
const SPL_TOKEN_ACCOUNT_RENT_VALUE = 5000000
```

La valeur precedente etait :

```ts
const SPL_TOKEN_ACCOUNT_RENT_VALUE = 2039280
```

Avec `2039280`, le `lz_receive` pouvait executer le mint, mais `PostExecute` echouait ensuite avec :

```text
InsufficientBalance
```

Apres rewiring avec `5000000`, les messages ont ete livres correctement.
