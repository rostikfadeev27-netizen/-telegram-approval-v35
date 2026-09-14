const TelegramBot = require('node-telegram-bot-api');

const TOKEN =
  process.env.TELEGRAM_BOT_TOKEN ||
  process.env.BOT_TOKEN ||
  process.env.TOKEN ||
  '';

const ALLOWED_USER_ID = Number(process.env.TELEGRAM_ALLOWED_USER_ID || '505975862');
const POLL_MS = Math.max(30000, Number(process.env.POLL_MS || '60000'));

if (!TOKEN) {
  console.error('ERROR: Telegram token is missing. Set TELEGRAM_BOT_TOKEN (or BOT_TOKEN/TOKEN).');
  process.exit(1);
}

const bot = new TelegramBot(TOKEN, { polling: true });

const USDT_TRC20 = 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t';

const routes = [
  {
    id: 'btc',
    label: 'Bitcoin (BTC)',
    source: 'bc1q3pzj73z6x99gjusq8gny3ycfkwg9h38nfs6d4x',
    destination: 'bc1q3klwywu4mve4na03qq5v994ydsqqvdp2an4lcn',
    asset: 'c0',
    decimals: 8,
    reserveBase: BigInt(process.env.BTC_FEE_RESERVE_SATS || '3000'),
    balance: getBtcBalance
  },
  {
    id: 'sol',
    label: 'Solana (SOL)',
    source: 'E9AFb7WJYdZKrrqibHWVqdrCKXVVvxzqUsu5ei1oBiV3',
    destination: 'Cu1rGmNykZg2sAhSBsPmTQDzbpH41LPQcXvbxeLqED2u',
    asset: 'c501',
    decimals: 9,
    reserveBase: BigInt(process.env.SOL_FEE_RESERVE_LAMPORTS || '10000'),
    balance: getSolBalance
  },
  {
    id: 'eth',
    label: 'Ethereum (ETH)',
    source: '0x9c57fabe6a3F9e8CdE5C28532C29358023b7F99f',
    destination: '0xd528D53c61032C781D78c6FbB463B52ab40C2a24',
    asset: 'c60',
    decimals: 18,
    reserveBase: BigInt(process.env.ETH_FEE_RESERVE_WEI || '300000000000000'),
    balance: () => getEvmBalance(
      process.env.ETH_RPC || 'https://ethereum-rpc.publicnode.com',
      '0x9c57fabe6a3F9e8CdE5C28532C29358023b7F99f'
    )
  },
  {
    id: 'bsc',
    label: 'BNB Smart Chain (BNB)',
    source: '0x9c57fabe6a3F9e8CdE5C28532C29358023b7F99f',
    destination: '0xd528D53c61032C781D78c6FbB463B52ab40C2a24',
    asset: 'c20000714',
    decimals: 18,
    reserveBase: BigInt(process.env.BSC_FEE_RESERVE_WEI || '300000000000000'),
    balance: () => getEvmBalance(
      process.env.BSC_RPC || 'https://bsc-rpc.publicnode.com',
      '0x9c57fabe6a3F9e8CdE5C28532C29358023b7F99f'
    )
  },
  {
    id: 'trx',
    label: 'TRON (TRX)',
    source: 'TDGZ4GfvCRe1d8oksj8fBD77ZHw4bkCPBA',
    destination: 'TLf9Nn6U2TVrTRWAqEuWX7xPxDpydd8wFn',
    asset: 'c195',
    decimals: 6,
    reserveBase: BigInt(process.env.TRX_FEE_RESERVE_SUN || '15000000'),
    balance: getTrxBalance
  },
  {
    id: 'usdt',
    label: 'USDT (TRC-20)',
    source: 'TDGZ4GfvCRe1d8oksj8fBD77ZHw4bkCPBA',
    destination: 'TLf9Nn6U2TVrTRWAqEuWX7xPxDpydd8wFn',
    asset: `c195_t${USDT_TRC20}`,
    decimals: 6,
    reserveBase: 0n,
    balance: getUsdtTrc20Balance
  }
];

const state = new Map(); // id -> { balance, initialized, error, lastCheck }

function allowed(userId) {
  return Number(userId) === ALLOWED_USER_ID;
}

function trimZeros(s) {
  if (!s.includes('.')) return s;
  return s.replace(/0+$/, '').replace(/\.$/, '');
}

function formatUnits(v, decimals) {
  const neg = v < 0n;
  let x = neg ? -v : v;
  const s = x.toString().padStart(decimals + 1, '0');
  const whole = s.slice(0, -decimals) || '0';
  const frac = decimals ? s.slice(-decimals) : '';
  const out = decimals ? trimZeros(`${whole}.${frac}`) : whole;
  return neg ? `-${out}` : out;
}

function amountForSend(route, delta) {
  let sendBase = delta;
  if (route.reserveBase > 0n) {
    sendBase = delta > route.reserveBase ? delta - route.reserveBase : delta;
  }
  return formatUnits(sendBase, route.decimals);
}

function trustSendUrl(route, amount) {
  const q = new URLSearchParams({
    asset: route.asset,
    address: route.destination,
    amount
  });
  return `https://link.trustwallet.com/send?${q.toString()}`;
}

async function fetchJson(url, options = {}) {
  const controller = new AbortController();
  const t = setTimeout(() => controller.abort(), 15000);
  try {
    const r = await fetch(url, { ...options, signal: controller.signal });
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return await r.json();
  } finally {
    clearTimeout(t);
  }
}

async function getBtcBalance() {
  const a = routes.find(x => x.id === 'btc').source;
  const j = await fetchJson(`https://mempool.space/api/address/${encodeURIComponent(a)}`);
  const confirmed = BigInt(j.chain_stats.funded_txo_sum) - BigInt(j.chain_stats.spent_txo_sum);
  const mempool = BigInt(j.mempool_stats.funded_txo_sum) - BigInt(j.mempool_stats.spent_txo_sum);
  return confirmed + mempool;
}

async function rpc(url, method, params) {
  const j = await fetchJson(url, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params })
  });
  if (j.error) throw new Error(j.error.message || 'RPC error');
  return j.result;
}

async function getEvmBalance(url, address) {
  const h = await rpc(url, 'eth_getBalance', [address, 'latest']);
  return BigInt(h);
}

async function getSolBalance() {
  const a = routes.find(x => x.id === 'sol').source;
  const r = await rpc(
    process.env.SOL_RPC || 'https://api.mainnet-beta.solana.com',
    'getBalance',
    [a, { commitment: 'confirmed' }]
  );
  return BigInt(r.value);
}

async function getTronAccount() {
  const a = routes.find(x => x.id === 'trx').source;
  const j = await fetchJson(`https://api.trongrid.io/v1/accounts/${encodeURIComponent(a)}`);
  return (j.data && j.data[0]) || {};
}

async function getTrxBalance() {
  const acc = await getTronAccount();
  return BigInt(acc.balance || 0);
}

async function getUsdtTrc20Balance() {
  const acc = await getTronAccount();
  const list = Array.isArray(acc.trc20) ? acc.trc20 : [];
  for (const obj of list) {
    if (obj && Object.prototype.hasOwnProperty.call(obj, USDT_TRC20)) {
      return BigInt(obj[USDT_TRC20] || '0');
    }
  }
  return 0n;
}

async function sendIncoming(route, delta, current) {
  const amount = amountForSend(route, delta);
  const url = trustSendUrl(route, amount);

  const text =
    `💰 Новое поступление\n\n` +
    `Сеть: ${route.label}\n` +
    `Получено: ${formatUnits(delta, route.decimals)}\n` +
    `Текущий баланс: ${formatUnits(current, route.decimals)}\n` +
    `Адрес назначения: ${route.destination}\n\n` +
    `Нажми кнопку ниже — откроется Trust Wallet с уже заполненными адресом и суммой. ` +
    `Перед отправкой Trust Wallet покажет комиссию и попросит подтверждение.`;

  await bot.sendMessage(ALLOWED_USER_ID, text, {
    reply_markup: {
      inline_keyboard: [[
        { text: '✅ Открыть Trust Wallet и подтвердить', url }
      ]]
    }
  });
}

async function checkOne(route, notify = true) {
  const s = state.get(route.id) || {};
  try {
    const bal = await route.balance();
    const now = new Date().toISOString();

    if (s.initialized && notify && bal > s.balance) {
      const delta = bal - s.balance;
      await sendIncoming(route, delta, bal);
    }

    state.set(route.id, {
      balance: bal,
      initialized: true,
      error: null,
      lastCheck: now
    });
  } catch (e) {
    state.set(route.id, {
      ...s,
      initialized: s.initialized || false,
      error: String(e.message || e),
      lastCheck: new Date().toISOString()
    });
    console.error(`[${route.id}]`, e.message || e);
  }
}

let checking = false;
async function checkAll(notify = true) {
  if (checking) return;
  checking = true;
  try {
    for (const r of routes) {
      await checkOne(r, notify);
    }
  } finally {
    checking = false;
  }
}

bot.onText(/^\/start$/, async (msg) => {
  if (!allowed(msg.from?.id)) return;
  await bot.sendMessage(
    msg.chat.id,
    `✅ Wallet Approval Bot работает.\n\n` +
    `Он автоматически проверяет BTC, SOL, ETH, BSC, TRX и USDT TRC-20.\n` +
    `При новом поступлении пришлёт кнопку, которая откроет Trust Wallet на экране отправки.\n\n` +
    `Команды:\n/status — текущие балансы\n/check — проверить сейчас\n/test — тестовые кнопки\n/reset — сбросить базовый баланс`
  );
});

bot.onText(/^\/check$/, async (msg) => {
  if (!allowed(msg.from?.id)) return;
  await bot.sendMessage(msg.chat.id, '🔄 Проверяю...');
  await checkAll(true);
  await bot.sendMessage(msg.chat.id, '✅ Проверка завершена. /status');
});

bot.onText(/^\/status$/, async (msg) => {
  if (!allowed(msg.from?.id)) return;
  const lines = ['📊 Статус мониторинга', ''];
  for (const r of routes) {
    const s = state.get(r.id);
    if (!s || !s.initialized) {
      lines.push(`${r.label}: ещё не проверено`);
    } else if (s.error) {
      lines.push(`${r.label}: ⚠️ ${s.error}`);
    } else {
      lines.push(`${r.label}: ${formatUnits(s.balance, r.decimals)}`);
    }
  }
  await bot.sendMessage(msg.chat.id, lines.join('\n'));
});

bot.onText(/^\/reset$/, async (msg) => {
  if (!allowed(msg.from?.id)) return;
  state.clear();
  await checkAll(false);
  await bot.sendMessage(msg.chat.id, '✅ Базовые балансы обновлены. Старые средства не будут считаться новым поступлением.');
});

bot.onText(/^\/test$/, async (msg) => {
  if (!allowed(msg.from?.id)) return;
  for (const r of routes) {
    const testBase = 10n ** BigInt(Math.max(0, r.decimals - 2));
    const amount = formatUnits(testBase, r.decimals);
    const url = trustSendUrl(r, amount);
    await bot.sendMessage(msg.chat.id, `🧪 ${r.label}\nТест открытия Trust Wallet`, {
      reply_markup: {
        inline_keyboard: [[{ text: `Открыть ${r.label}`, url }]]
      }
    });
  }
});

bot.on('polling_error', e => {
  console.error('Telegram polling error:', e.message || e);
});

(async () => {
  console.log(`Wallet Approval Bot started. User ID: ${ALLOWED_USER_ID}`);
  await checkAll(false);
  setInterval(() => checkAll(true), POLL_MS);
})();
