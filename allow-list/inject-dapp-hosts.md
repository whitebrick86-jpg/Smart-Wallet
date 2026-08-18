# Inject / dApp allow list (content scripts)

Apex hosts where Smart Wallet may **inject** the page provider (Wallet Standard / EIP-1193).
Subdomains of each apex are included (e.g. `app.uniswap.org` under `uniswap.org`).

**Source of truth (extension):** `inject-allowlist.js` — snapshot synced **2026-08-18** for product **0.11.298**.
**Copy in this folder:** `inject-allowlist-source.js`

**Production does not inject on** `localhost` or `127.0.0.1`.

---

**Total apex hosts:** 105

## Full list

| # | Host |
|---|------|
| 1 | jup.ag |
| 2 | jupiter.ag |
| 3 | pump.fun |
| 4 | raydium.io |
| 5 | orca.so |
| 6 | tensor.trade |
| 7 | drift.trade |
| 8 | mango.markets |
| 9 | kamino.finance |
| 10 | sanctum.so |
| 11 | sol-incinerator.com |
| 12 | meteora.ag |
| 13 | phoenix.trade |
| 14 | birdeye.so |
| 15 | dexscreener.com |
| 16 | photon-sol.tinyastro.io |
| 17 | bullx.io |
| 18 | trojan.com |
| 19 | fluxbot.xyz |
| 20 | bonkbot.io |
| 21 | jito.network |
| 22 | four.meme |
| 23 | uniswap.org |
| 24 | pancakeswap.finance |
| 25 | sushi.com |
| 26 | sushiswap.com |
| 27 | 1inch.io |
| 28 | 1inch.exchange |
| 29 | matcha.xyz |
| 30 | 0x.org |
| 31 | curve.fi |
| 32 | balancer.fi |
| 33 | dodoex.io |
| 34 | kyberswap.com |
| 35 | paraswap.io |
| 36 | cow.fi |
| 37 | aerodrome.finance |
| 38 | velodrome.finance |
| 39 | traderjoexyz.com |
| 40 | quickswap.exchange |
| 41 | spooky.fi |
| 42 | camelot.exchange |
| 43 | thena.fi |
| 44 | biswap.org |
| 45 | apeswap.finance |
| 46 | odos.xyz |
| 47 | openocean.finance |
| 48 | bebop.xyz |
| 49 | gmx.io |
| 50 | hyperliquid.xyz |
| 51 | dydx.exchange |
| 52 | dydx.trade |
| 53 | hashflow.com |
| 54 | woofi.com |
| 55 | woo.org |
| 56 | thruster.finance |
| 57 | baseswap.fi |
| 58 | swapbased.finance |
| 59 | alienbase.xyz |
| 60 | equalizer.exchange |
| 61 | zyberswap.io |
| 62 | arbidex.fi |
| 63 | ramses.exchange |
| 64 | maverick.xyz |
| 65 | mav.xyz |
| 66 | frax.finance |
| 67 | shibaswap.com |
| 68 | bancor.network |
| 69 | airswap.io |
| 70 | defillama.com |
| 71 | llama.fi |
| 72 | krystal.app |
| 73 | web3.okx.com |
| 74 | relay.link |
| 75 | li.fi |
| 76 | jumper.exchange |
| 77 | stargate.finance |
| 78 | across.to |
| 79 | socket.tech |
| 80 | bungee.exchange |
| 81 | debridge.finance |
| 82 | orbiter.finance |
| 83 | hop.exchange |
| 84 | synapseprotocol.com |
| 85 | cbridge.celer.network |
| 86 | portalbridge.com |
| 87 | wormhole.com |
| 88 | base.org |
| 89 | arbitrum.io |
| 90 | optimism.io |
| 91 | avax.network |
| 92 | polygon.technology |
| 93 | bscscan.com |
| 94 | etherscan.io |
| 95 | basescan.org |
| 96 | polygonscan.com |
| 97 | arbiscan.io |
| 98 | optimistic.etherscan.io |
| 99 | snowtrace.io |
| 100 | aave.com |
| 101 | compound.finance |
| 102 | opensea.io |
| 103 | blur.io |
| 104 | robinhood.com |
| 105 | ens.domains |

## Notes

- Manifest `content_scripts.matches` and `web_accessible_resources.matches` must cover the same sites (generated via extension tools when hosts change).
- Content scripts load: `inject-allowlist.js` → `ui/ui-logo.js` → `content-script.js`.
- Chrome Web Store / chrome.google.com are **blocked** for inject.
