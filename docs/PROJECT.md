# Projet: Token multi-chaînes — Ethereum (Sepolia), Solana, LayerZero OFT

## Objectif

Ce projet vise à:

- Déployer un token ERC20 sur le testnet Sepolia (Ethereum).
- Créer un token natif sur Solana et déléguer les autorités de mint et burn à un multisig à 2 comptes.
- Implémenter un OFT (Omnichain Fungible Token) via LayerZero pour permettre le transfert inter-chaînes.

## Résumé technique

- ERC20: contrat basé sur OpenZeppelin `ERC20`.
- Solana: mint SPL token; multisig contrôlera mint & burn.
- LayerZero: suivre l'exemple OFT (LayerZero docs) pour envelopper le token et permettre le bridging.

---

## 1) ERC20 — Sepolia

### Contrat (exemple minimal)

Fichier: `contracts/Token.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract TokenzExample is ERC20 {
    constructor(uint256 initialSupply) ERC20("TokenzExample", "TZX") {
        _mint(msg.sender, initialSupply);
    }
}
```

### Déploiement (exemple Hardhat)

- Installer: `npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers @openzeppelin/contracts`
- Configurer `hardhat.config.js` avec une URL RPC Sepolia (Alchemy/Infura) et la clé privée du deployer (ne pas committer).
- Script de déploiement simple (`scripts/deploy.js`) : déployer `TokenzExample` et envoyer le total supply désiré.

Remarques:

- Pour Sepolia, utilisez une clé privée de test et HDRP (Alchemy/Infura) ou `ethrpc` public.
- Vérifiez le contrat sur Etherscan (si désiré) après le déploiement.

---

## 2) Solana — mint + multisig

### Objectif

Créer un mint SPL et donner les autorités de `mint` et `burn` à un multisig contrôlé par 2 comptes (seuil 2/2).

### Étapes générales (haute-niveau)

1. Générer deux paires de clés (ou utiliser wallets existants).
2. Créer le multisig on-chain (avec `spl-token create-multisig` ou via programme JS).
3. Créer le mint (ex: `spl-token create-token` ou `spl-token create-mint`) par un deployer.
4. Mettre à jour les autorités du mint: `spl-token authorize <MINT> mint <MULTISIG_ADDRESS>` et `spl-token authorize <MINT> freeze <MULTISIG_ADDRESS>` (ou commandes équivalentes pour burn).

Notes pratiques:

- Vous pouvez utiliser la CLI `spl-token` ou `@solana/spl-token` (JS) / Anchor pour automatiser.
- Conserver en sécurité les clés des 2 signataires; le multisig nécessitera les 2 signatures pour mint/burn.

---

## 3) LayerZero — OFT (Omnichain Fungible Token)

### But

Permettre au token d'être transféré entre chaînes via LayerZero en créant un OFT wrapper ou en adaptant le token selon les exemples LayerZero.

### Étapes proposées

1. Lire la documentation LayerZero OFT: https://layerzero.gitbook.io/ (consulter la section OFT / token bridging).
2. Choisir l'implémentation: déployer l'OFT contract (souvent étend ERC20 + LayerZero messaging) sur Ethereum Sepolia.
3. Configurer `lzEndpoint` pour Sepolia et pour les chainIds cibles.
4. Tester les transferts inter-chaînes en local / testnet entre Sepolia <-> autre réseau supporté (ou un mock).

Remarques:

- LayerZero requiert la configuration des endpoints et parfois des frais pour messaging adapters.
- Suivre les exemples officiels et scripts de déploiement.

---

## Checklist & livrables

- [ ] Contrat ERC20 complet et audité sommairement.
- [ ] Script Hardhat pour déployer sur Sepolia.
- [ ] Mint Solana créé et multisig 2/2 configuré pour mint/burn.
- [ ] Documentation OFT et contrat déployé/configuré.
- [ ] Tests end-to-end (mint sur Solana, lock+mint sur OFT, transfert inter-chaînes).

## Prochaines tâches (suggestions)

- Générer le contrat ERC20 final (nom/Ticker/totalSupply) et le script de déploiement.
- Créer les keypairs Solana et script `scripts/solana-setup.sh` pour automatiser la création du multisig et l'affectation d'autorités.
- Lire et suivre les guides LayerZero pour créer et déployer l'OFT; préparer tests.

---

## Notes et liens utiles

- OpenZeppelin ERC20: https://docs.openzeppelin.com/
- Solana SPL Token CLI: https://spl.solana.com/token
- LayerZero docs: https://layerzero.gitbook.io/

---

Si tu veux, je peux maintenant:

- Générer le contrat ERC20 complet et le script Hardhat, ou
- Écrire les scripts Solana CLI/JS pour créer le multisig et définir les autorités, ou
- Chercher précisément la doc LayerZero et préparer le contrat OFT et scripts de déploiement.

Dis-moi quelle action tu veux que je fasse en premier.
