// === BSC New Pair Alert Bot ===
// Author: ChatGPT + Ibezimako
// Description: Scans Dexscreener for new BSC pairs and sends Telegram alerts

import { Telegraf } from "telegraf";
import axios from "axios";

// === YOUR BOT INFO (Direct Embed) ===
const BOT_TOKEN = "8388984179:AAFMf_A4_ehHffdKxgES-EmNy3Npyx1yA3w";
const CHAT_ID = "5570281457";
const bot = new Telegraf(BOT_TOKEN);

// === CONFIG ===
const CHECK_INTERVAL = 30 * 1000; // 30 seconds
const MIN_LIQUIDITY = 10000;
const MIN_VOLUME = 100000;
const MAX_TAX = 5;
const MAX_AGE_MINUTES = 60; // under 1 hour

let seenPairs = new Set();

// === MAIN FUNCTION ===
async function checkNewPairs() {
  try {
    const res = await axios.get("https://api.dexscreener.com/latest/dex/pairs/bsc");
    const pairs = res.data.pairs || [];

    for (const pair of pairs) {
      const {
        pairAddress,
        baseToken,
        priceUsd,
        liquidity,
        volume,
        fdv,
        info,
        pairCreatedAt,
      } = pair;

      if (!pairAddress || seenPairs.has(pairAddress)) continue;
      seenPairs.add(pairAddress);

      // --- Filter ---
      const liquidityUsd = liquidity?.usd || 0;
      const volumeUsd = volume?.h24 || 0;
      const marketCap = fdv || 0;
      const ageMinutes = (Date.now() - new Date(pairCreatedAt).getTime()) / 60000;

      if (
        liquidityUsd < MIN_LIQUIDITY ||
        volumeUsd < MIN_VOLUME ||
        ageMinutes > MAX_AGE_MINUTES
      ) continue;

      // Optional verified + honeypot filter placeholders
      const isVerified = info?.verified || false;
      const honeypotCheck = "✅"; // Placeholder - replace with external API check if desired
      const logo = baseToken?.image || null;

      // --- Format alert ---
      const message = `
🚀 <b>New BSC Pair</b>
Token: <b>$${baseToken.symbol}</b>
Market Cap: <b>$${(marketCap / 1000).toFixed(1)}k</b>
Volume: <b>$${(volumeUsd / 1000).toFixed(1)}k</b>
Liquidity: <b>$${(liquidityUsd / 1000).toFixed(1)}k</b>
Age: <b>${ageMinutes.toFixed(1)} min</b>
Verified: ${isVerified ? "✅" : "❌"}
Honeypot: ${honeypotCheck}
Contract: <code>${baseToken.address}</code>
Dexscreener: https://dexscreener.com/bsc/${pairAddress}
`;

      if (logo) {
        await bot.telegram.sendPhoto(CHAT_ID, logo, {
          caption: message,
          parse_mode: "HTML",
        });
      } else {
        await bot.telegram.sendMessage(CHAT_ID, message, {
          parse_mode: "HTML",
        });
      }

      console.log(`✅ Sent alert for ${baseToken.symbol}`);
    }
  } catch (err) {
    console.error("Error fetching pairs:", err.message);
  }
}

// === START LOOP ===
setInterval(checkNewPairs, CHECK_INTERVAL);
console.log("🚀 BSC Alert Bot running...");
bot.launch();
