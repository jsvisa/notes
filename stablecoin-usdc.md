# The design of Circle USDC

## Overview

- Deployed in 15 blockchains, https://www.circle.com/en/multi-chain-usdc
- Ethereum mainnet deployed on 2018-08-03, https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 (USDT 2017-11-28)
- EVM contract code: https://github.com/circlefin/stablecoin-evm

## Contracts

### USDC

USDC: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48

USDC is following [OpenZeppelin standards](https://docs.openzeppelin.com/contracts/2.x/):

- [Upgradeable](https://docs.openzeppelin.com/contracts/5.x/api/proxy): use `org.zeppelinos.proxy.implementation` Proxy
- [Ownable](https://docs.openzeppelin.com/contracts/2.x/api/ownership#Ownable)
- [Pausable](https://docs.openzeppelin.com/contracts/4.x/api/security#Pausable)

And USDC is also [Blacklistable](https://github.com/centrehq/centre-tokens/blob/master/contracts/v1/Blacklistable.sol), when an address was blacklisted, it can't do any asset related operations:

- `mint`: msg.sender, to
- `burn`: msg.sender
- `transfer{From}``: msg.sender, from, to
- `approve`: msg.sender, spender

#### Roles

- admin: swap between 0x807a96288A1A408dBC13DE2b1d087d10356395d2(EOA) and UpgradeContract
  - upgrade contract

- owner: 0xfcb19e6a322b27c06842a71e8c725399f049ae3a(EOA)
  - change blacklister
  - change pauser
  - change masterMinter

- pauser: 0x4914f61d25e5c567143774b76edbf4d5109a8566(EOA)
  - pause/unpause

- blacklister: 0x10df6b6fe66dd319b1f82bab2d054cbb61cdad2e(EOA)
  - blacklist/unblacklist

- masterMinter: 0xe982615d461dd5cd06575bbea87624fda4e3de17(Contract)
  - addMinter/removeMinter
  - configure minter minting allowance

### Upgrade

Upgraded(address) '0xbc7cd75a20ee27fd9adebab32041f755214dbc6bffa90cc0225b39da2e5c2d3b'

upgrade process:

1. [Circle Deployer](https://etherscan.io/address/0x95Ba4cF87D6723ad9C0Db21737D862bE80e93911) create a new Upgrader contract;
2. [USDC Admin](https://etherscan.io/address/0x807a96288A1A408dBC13DE2b1d087d10356395d2) call `changeAdmin` on USDC to change the admin from self to the new Upgrader contract;
3. [Circle Deployer](https://etherscan.io/address/0x95Ba4cF87D6723ad9C0Db21737D862bE80e93911) call `upgrade` on the Upgrader contract to upgrade the USDC contract, in the meanwhile, the admin of USDC will be changed back to the EOA address

eg:

1. 18921600 https://etherscan.io/tx/0x7f6268ff5bd05d1b61c19889a46eb9a38563accce441dcfcf0c7515b1733503e create the upgrade contract
2. 18963710 https://etherscan.io/tx/0x4441f5f0d30b59f9db3083034605e5cfb35e3606a30e12f075feef3b1df81a15 change admin to the upgrade contract
3. 18963716 https://etherscan.io/tx/0xae3ad89e569f27d47a8a02999a6d937c12aaa6bc50e66650e7cbd3244bde9951 upgrading

### USDC Master Minter

The Master Minter contract manages minters for a contract, It lets the owner designate certain addresses as controllers,
and these controllers then manage the minters by adding and removing minters, as well as modifying their minting allowance.

**A controller may manage exactly one minter, but the same minter address may be managed by multiple controllers.**

USDC Master Minter: 0xe982615d461dd5cd06575bbea87624fda4e3de17

- owner: 0xc1d9fe41d19dd52cb3ae5d1d3b0030b5d498c704(EOA)
- mintManager: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48(USDC Token)

#### workflow

eg: increase minter's allowance

1. owner.register (controller + minter): `configureController(address _controller, address _worker) public onlyOwner`
2. controller.increase minter allowance: `incrementMinterAllowance(uint256 _allowanceIncrement) public onlyController`
   2.1. get minter with controller's address
   2.2. set `_newAllowance = _allowanceIncrement + minterManager.minterAllowance(minter)`
   2.2. call `minterManager.configureMinter(_minter, _newAllowance)`

## Data

1. All USDC events(2018 -- 2024.09.11)

```sql
+-----------------------+----------+-------------+-------------+
| event                 | count    | min_blktime | max_blktime |
|-----------------------+----------+-------------+-------------|
| Transfer              | 99445585 | 2018-09-10  | 2024-09-11  |
| Approval              | 10130649 | 2018-09-28  | 2024-09-11  |
| Burn                  | 281194   | 2018-09-25  | 2024-09-11  |
| Mint                  | 205210   | 2018-09-10  | 2024-09-11  |
| AuthorizationUsed     | 8144     | 2020-09-27  | 2024-09-10  |
| MinterConfigured      | 1056     | 2018-08-17  | 2024-09-09  |
| Blacklisted           | 241      | 2020-06-16  | 2024-09-06  |
| AdminChanged          | 8        | 2018-08-03  | 2024-01-08  |
| UnBlacklisted         | 6        | 2021-12-23  | 2024-02-06  |
| MinterRemoved         | 4        | 2018-09-06  | 2021-04-06  |
| Upgraded              | 3        | 2020-08-27  | 2024-01-08  |
| PauserChanged         | 3        | 2018-09-06  | 2023-08-21  |
| AuthorizationCanceled | 3        | 2021-04-02  | 2024-01-08  |
| MasterMinterChanged   | 2        | 2018-09-06  | 2019-06-12  |
| BlacklisterChanged    | 2        | 2019-06-27  | 2023-08-21  |
| OwnershipTransferred  | 1        | 2018-09-06  | 2018-09-06  |
+-----------------------+----------+-------------+-------------+
```

2. USDC Upgrade history

```csv
date,txhash,implementation
2024-01-08,0xae3ad89e569f27d47a8a02999a6d937c12aaa6bc50e66650e7cbd3244bde9951,0x43506849d7c04f9138d1a2050bbf3a0c054402dd
2021-04-26,0xe2e40640ffd5f76538cd23660cf56f00bfebd5fe925ebad6b8067c4cee18a2c3,0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf
2020-08-27,0xe6f0f754398d89583da8e4229c5d7aaa00739a3ae334ecfc2839ac396b4836e3,0xb7277a6e95992041568d9391d09d0122023778a2
```

Reference:

- https://github.com/bhemen/usdc/blob/main/README.md#v22-upgrade-january-2024
- https://github.com/bhemen/usdc/blob/main/README.md#v21-upgrade
- https://github.com/bhemen/usdc/blob/main/README.md#v2-upgrade-december-2020

3. USDC Minters

```csv
minter,is_contract,allowance
0x5b6122c109b78c6755486966148c1d70a50a47d7,false,2648313326
0xc4922d64a24675e16e1586e3e3aa56c06fabe907,true,27392554
0x19a932fc5a8320939c3575302a8705147a7f27d8,false,24198
0x55fe002aeff02f77364de339a1292923a15844b8,false,0
0x24bdd8771b08c2ea6fe0e898126e65bd49021be3,false,0
0x3005a4c0efe7e66f3f60ef8704983247a5c6ca61,false,0
0x8967a7ce20043f876e42f8ad696b06bb632f0ca7,false,0
0x895f07957b863f4ab6086035a6990d8366bc3266,false,0
```

4. USDC Master Minter Controllers

`event ControllerConfigured(address indexed _controller, address indexed _worker)`
'0xa56687ff5096e83f6e2c673cda0b677f56bbfcdf5fe0555d5830c407ede193cb'

```csv
minter,controller
0x5b6122c109b78c6755486966148c1d70a50a47d7,0x9d5c50ae9dc377b1fde6786bcfe8a70854fac5e4
0x5b6122c109b78c6755486966148c1d70a50a47d7,0x79e0946e1c186e745f1352d7c21ab04700c99f71
0xc4922d64a24675e16e1586e3e3aa56c06fabe907,0x0f493479be830657aca110199a70e826506a54d3
0x19a932fc5a8320939c3575302a8705147a7f27d8,0x4024071fe3fad805f922a9098630d09a2ec0f82c
0x19a932fc5a8320939c3575302a8705147a7f27d8,0xb0c9f95c1f0e686249ae0461e5e7657545379017
0xe7ab0dd2a069fa115c0d7878af6fd95ba0f9100a,0x33c1b799978d3f7830a24c6eddd169caa4b92b61
0xd4c1315948125cd20c11c5e9565a3632c1710055,0xfc48855a426c59d3cc9efd95ec3de1fcf6b2790e
0x8967a7ce20043f876e42f8ad696b06bb632f0ca7,0x14b1bc91cb99aad649b36341d5abeb50df775a3e
0x8967a7ce20043f876e42f8ad696b06bb632f0ca7,0xeb10024c7761e78cbbe78773ac6f6e98f3ed8314
0x895f07957b863f4ab6086035a6990d8366bc3266,0x08c71ca483e6c454a35d2ed56073a0297f0a2ec3
0x55fe002aeff02f77364de339a1292923a15844b8,0x14b1bc91cb99aad649b36341d5abeb50df775a3e
0x3005a4c0efe7e66f3f60ef8704983247a5c6ca61,0x08c71ca483e6c454a35d2ed56073a0297f0a2ec3
0x3005a4c0efe7e66f3f60ef8704983247a5c6ca61,0x7b3d54052df209679fce721516cc1b566ef1c6ab
0x24bdd8771b08c2ea6fe0e898126e65bd49021be3,0x39a372b4cc3fc91ba93238c9121cb4f98d057fb5
0x24bdd8771b08c2ea6fe0e898126e65bd49021be3,0x38d51f44b586b154159932f3681a30c54a6d25a9
```