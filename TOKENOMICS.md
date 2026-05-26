# DevSwap Token — `$DSWP` (كل ما يخص العملة)

> وثيقة جامعة لعملة المنصّة `$DSWP`: ماهيّتها، عقدها، اقتصادها، آلية قيمتها، توزيعها،
> تشغيلها، وإطارها القانوني الآمن. **كل رقم/سلوك هنا مرجوع للكود المصدري** (لا تقديرات).
>
> مصادر الحقيقة: [`STATE.md`](../STATE.md) > [`PLAN.md`](../PLAN.md) > الكود. عند أي تعارض، الكود هو الفصل.
> آخر تحديث: 2026-05-23 · الشبكة: BSC · التسوية: USDT · العملة: `$DSWP`.

---

## 0) ملخّص تنفيذي (TL;DR)

- `$DSWP` عملة **منفعة (utility)** قياسية ERC-20 على BSC — **ليست عملة دفع** ولا أداة استثمار.
- الدفع والضمان يتمّان بالكامل بـ**USDT**. مهمّة `$DSWP` الوحيدة: **يقلّ معروضها** مع نشاط المنصّة.
- **المعروض الأقصى: 100,000,000** (سقف صلب لا يُتجاوز).
- **آلية القيمة:** كل صفقة ناجحة توجّه **1.5%** لشراء `$DSWP` من PancakeSwap وحرقها (buyback-and-burn) → انكماش مرتبط بـGMV.
- **قرار استراتيجي مثبّت:** العملة **مؤجّلة** حتى يُثبَت الطلب (validate-first) — يُطلق MVP بعمولة USDT أولاً.
- **إطار قانوني آمن (securities-safe):** لا staking/yield/farming، ولا أي وعد بارتفاع السعر.

> **طلب العملة (مستقبلي، بعد الإطلاق):** تصميم المصارف الاستهلاكية (رفع ظهور المطوّر + أولوية طلب العميل +
> باقات التسويق مع باقة مجانية دائمة) موثّق في [`TRUST-AND-INCENTIVES.md` — Area C](TRUST-AND-INCENTIVES.md):
> رفع = مضاعِف على الجدارة لا بديل عنها (مسقوف، مُعلَّم «مُموَّل»، أرضية سمعة)، والمصروف **يُحرَق آلياً** ولا
> يُسوَّق الحرق كعائد. (ADR-0016 محجوز.)

---

## 1) ما هي `$DSWP` وما ليست به

| هي | ليست |
|---|---|
| عملة منفعة ERC-20 (BEP-20) | عملة دفع/تسوية (الدفع بـUSDT) |
| أداة انكماشية عبر الحرق | أداة استثمار/عائد |
| قابلة للحرق (`burn()`) | غير قابلة للسكّ فوق السقف |
| ملكيتها `Ownable2Step` (تصبح multisig) | مملوكة بمفتاح مفرد على mainnet |

**لماذا لا تُستخدم في الدفع؟** ثبات السعر للطرفين (عميل/مطوّر) يتطلب عملة مستقرّة → USDT.
استخدام عملة متقلّبة في الضمان يعرّض الطرفين لمخاطر السعر أثناء العمل. لذا فُصلت الوظيفتان عمداً:
**الاستقرار للتسوية (USDT)** و**الانكماش للعملة (`$DSWP`)**.

---

## 2) عقد التوكن — [`DevSwapToken.sol`](../contracts/src/DevSwapToken.sol)

عقد بسيط متعمَّد (34 سطراً)، Solidity `=0.8.24`، OpenZeppelin v5.1.0 (vendored).

```solidity
contract DevSwapToken is ERC20, ERC20Burnable, ERC20Capped, Ownable2Step
```

### المواصفات

| الخاصية | القيمة | الغرض |
|---|---|---|
| الاسم / الرمز | `DevSwap` / `DSWP` | تعريف ERC-20 |
| الخانات (decimals) | 18 | افتراضي ERC-20 |
| السقف المطلق `MAX_SUPPLY` | `100_000_000e18` | سقف صلب على `totalSupply` |
| `ERC20Capped` | — | `totalSupply` **لا يتجاوز السقف أبداً** حتى بعد الحرق وإعادة السكّ |
| `ERC20Burnable` | — | يُمكّن الـescrow من حرق ما اشتراه عبر `burn()` |
| `Ownable2Step` | — | نقل ملكية بخطوتين (يمنع الإرسال لعنوان خاطئ) |

### الدوال

| الدالة | الصلاحية | السلوك |
|---|---|---|
| `constructor(initialOwner)` | — | يضبط الاسم/الرمز/السقف والمالك. **لا يسكّ أي توكن** — `totalSupply` تبدأ صفراً |
| `mint(to, amount)` | `onlyOwner` | يسكّ حتى السقف؛ يفشل عبر `ERC20Capped` إن تجاوزه |
| `_update(...)` | `internal override` | يحلّ تعارض الوراثة الماسي بين `ERC20` و`ERC20Capped` |

> **نقطة جوهرية:** الحرق يتم عبر `ERC20Burnable.burn()` على عملات يملكها العقد — **لا** تحويل لـ`address(0)`
> (OpenZeppelin ERC20 يفشل على التحويل لعنوان صفري). راجع [`PLAN.md §⚠️2`](../PLAN.md).

---

## 3) اقتصاديات العملة (Tokenomics)

**المعروض الأقصى: 100,000,000 `$DSWP`** — موزّع على 4 فئات. الأرقام مطابقة لثوابت سكربت التوزيع
[`DistributeToken.s.sol`](../contracts/script/DistributeToken.s.sol):

| # | الفئة | النسبة | الكمية | الوجهة (env) | الآلية |
|---|---|---:|---:|---|---|
| 1 | تعدين النشاط والتبادل | **50%** | 50,000,000 | `ACTIVITY_DISTRIBUTOR` (افتراضي `TREASURY`) | حوافز للمطوّرين النشطين |
| 2 | مجمع السيولة (الإطلاق) | **25%** | 25,000,000 | `TREASURY` | قفل LP على PancakeSwap مقابل USDT |
| 3 | الفريق والتطوير | **15%** | 15,000,000 | `VestingWallet(TEAM_BENEFICIARY)` | **Vesting خطّي** (افتراضي 4 سنوات) |
| 4 | المكافآت المجتمعية (Airdrop) | **10%** | 10,000,000 | `AIRDROP_DISTRIBUTOR` | لمطوّري GitHub النشطين |
| | **الإجمالي** | **100%** | **100,000,000** | | السكربت يؤكّد `totalSupply == 100M` |

### Vesting الفريق

- يُسكّ نصيب الفريق (15M) إلى عقد **OpenZeppelin `VestingWallet`** خطّي.
- البدء: `TEAM_VEST_START` (افتراضي وقت النشر) · المدة: `TEAM_VEST_DURATION` (افتراضي `4 * 365 days`).
- الإفراج خطّي طوال المدة — لا cliff في التهيئة الحالية (قابل للتعديل عبر env قبل النشر).

---

## 4) آلية القيمة — Buyback & Burn (قلب الاقتصاد)

العملة لا تكتسب قيمتها من وعد سعري، بل من **انكماش مرتبط بالنشاط الفعلي**:

```
صفقة ناجحة (USDT) ──► 97% للمطوّر + 1.5% للمالك (فوراً)
                              │
                              └─► 1.5% USDT ──► شراء $DSWP من PancakeSwap ──► burn() ──► معروض أقل
```

### حساب التوزيع (دالة `_payout`)

على USDT بـ**18 خانة** (مهم: BSC USDT 18 خانة، لا 6 كـEthereum):

```solidity
fee          = amount * FEE_BPS     / BPS_DENOMINATOR   // 150/10000 = 1.5%
buyback      = amount * BUYBACK_BPS / BPS_DENOMINATOR   // 150/10000 = 1.5%
developerNet = amount - fee - buyback                   // 97% (الباقي يمنع ضياع dust)
```

ثوابت العقد: `FEE_BPS = 150` · `BUYBACK_BPS = 150` · `BPS_DENOMINATOR = 10_000`.
الإجمالي **3% فقط** — والمطوّر يحتفظ بـ**97%**.

### مساران للحرق (Option C — قرار المالك المثبّت في [`STATE.md`](../STATE.md))

**(أ) فوري داخل `releaseFunds` (auto-buyback):**
1. يُدفع للمطوّر (97%) والمالك (1.5%) **أولاً** — أموالهم لا تعتمد على السوق إطلاقاً.
2. ثم محاولة حرق الـ1.5% فوراً عبر self-call `try this.autoBuybackAndBurn(buyback)`.
3. التسعير on-chain: `getAmountsOut` ثم `minOut = expectedOut * (10000 - buybackSlippageBps) / 10000`.
4. **عند فشل السواب** (سيولة ناقصة/انزلاق/مهلة): `catch` يؤجّل المبلغ إلى `buybackReserve` ويُصدر `BuybackDeferred` — والمطوّر مدفوع مسبقاً.

**(ب) مجمّع لاحق `executeBuybackBurn(minDswpOut, deadline)`:**
- المالك أو الـkeeper يحرق `buybackReserve` المتراكم دفعةً واحدة.
- **CEI:** تصفير `buybackReserve` **قبل** السواب؛ فشل السواب يعكس المعاملة كاملة فيُستعاد الرصيد.
- `forceApprove` للراوتر ثم `swapExactTokensForTokens([USDT,DSWP])` ثم `dswp.burn(received)`.

> **لماذا الفصل؟** فشل السوق (بركة جافة، انزلاق عالٍ) **يجب ألّا يحجب أجر المطوّر**.
> الدفع للبشر أولاً، والحرق محاولة معزولة قابلة للتأجيل. هذا أهم قرار أمني-اقتصادي في النظام.

### ضبط الانزلاق (Slippage / MEV)

| الموضع | المتغيّر | الافتراضي | السقف |
|---|---|---|---|
| الحرق الفوري (داخل العقد) | `buybackSlippageBps` | 300 (3%) | `MAX_SLIPPAGE_BPS = 1000` (10%) |
| الـkeeper (المجمّع، off-chain) | `SLIPPAGE_BPS` | 100 (1%) | يحدّده المشغّل |

كلا المسارين يستخدمان `amountOutMin` + `deadline` لتقليل sandwich-attacks والتعبئة السيئة.
التجميع الدوري يقلّل الأثر السعري وكلفة الغاز للمعاملة الفردية.

---

## 5) التوزيع على السلسلة — [`DistributeToken.s.sol`](../contracts/script/DistributeToken.s.sol)

سكربت Foundry ينشر `$DSWP` ويسكّ كامل الـ100M وفق الجدول، ثم يسلّم الملكية:

1. الناشر ينشر العقد ويملكه مؤقتاً (ليتمكّن من السكّ).
2. ينشئ `VestingWallet` للفريق.
3. يسكّ: 50M نشاط · 25M سيولة · 15M فريق (للـvesting) · 10M airdrop.
4. ينقل الملكية إلى `ESCROW_OWNER` عبر `transferOwnership` (2-step — المالك يستدعي `acceptOwnership()`).
5. يؤكّد `totalSupply() == 100M` (يفشل النشر إن اختلّ المجموع).

**متغيّرات بيئية مطلوبة:** `PRIVATE_KEY`, `TREASURY`, `AIRDROP_DISTRIBUTOR`, `TEAM_BENEFICIARY`.
**اختيارية:** `ESCROW_OWNER` (افتراضي الناشر), `ACTIVITY_DISTRIBUTOR` (افتراضي `TREASURY`),
`TEAM_VEST_START` (افتراضي الآن), `TEAM_VEST_DURATION` (افتراضي 4 سنوات).

> على mainnet: `ESCROW_OWNER` يجب أن يكون **multisig (Gnosis Safe 3-of-5)** + timelock — راجع بوابة P5.

---

## 6) الـKeeper — تشغيل الحرق المجمّع off-chain ([`keeper/src/index.ts`](../keeper/src/index.ts))

بوت TypeScript خفيف (viem) يشغّل المسار المجمّع (ب):

1. يقرأ `buybackReserve()` من الـescrow.
2. إن كان دون `MIN_RESERVE_WEI` → يتخطّى (لتجنّب حرق ضئيل مكلف غازياً).
3. يحسب `expectedOut` عبر `getAmountsOut`، ثم `minOut` بعد `SLIPPAGE_BPS` (افتراضي 1%).
4. `deadline = now + DEADLINE_SECS` (افتراضي 300 ثانية).
5. يرسل `executeBuybackBurn(minOut, deadline)` وينتظر الإيصال.
6. وضع التكرار: إن وُجد `INTERVAL_MS` > 0 يعمل في حلقة دورية؛ وإلّا تشغيلة واحدة.

**متغيّرات:** `RPC_URL`, `PRIVATE_KEY`, `ESCROW_ADDRESS`, `USDT_ADDRESS`, `DSWP_ADDRESS`, `ROUTER_ADDRESS`,
+ اختيارية `SLIPPAGE_BPS`, `MIN_RESERVE_WEI`, `DEADLINE_SECS`, `INTERVAL_MS`, `CHAIN_ID` (56=mainnet, 97=testnet).

> الـkeeper مكمّل لا بديل: الحرق الفوري يغطّي الحالة العادية، والـkeeper يحرق المؤجّل (الفاشل/المعطّل).
> صلاحية الـkeeper محدودة بـ`executeBuybackBurn` فقط — لا يمسّ أموال المهام.

---

## 7) السيولة (Liquidity)

- **25M `$DSWP`** مخصّصة لبركة `DSWP/USDT` على **PancakeSwap V2**.
- الشراء التلقائي يتطلّب **بركة عميقة** — مبكراً = أثر سعري عالٍ. لذا يبدأ التجميع بعد توفّر سيولة كافية ([`PLAN.md §⚠️4`](../PLAN.md)).
- **رأس المال المقابل:** $250k–$1.25M (قرار مالك مفتوح — [`STATE.md`](../STATE.md)).
- **قفل LP:** عبر **PinkLock** بعد إضافة السيولة (دون قفل = إشارة scam في عيون السوق).
- Router: PancakeSwap V2 `0x10ED43C718714eb63d5aA57B78B54704E256024E`.

---

## 8) الإطار القانوني الآمن (Securities-Safe Framing)

تُصاغ العملة عمداً لتقليل التعرّض التنظيمي (يطابق معيار securities-safe المعتمد):

- ❌ **ممنوع:** staking · yield · farming · vaults · أي وعد بعائد أو ارتفاع سعر.
- ✅ **مسموح:** وصف الانكماش كآلية تقنية مرتبطة بالنشاط، مع التصريح "أثر السعر يعتمد على ظروف السوق ولا يُضمَن".
- **سياسة لغة آمنة (`docs` + كود):** كلمة `escrow`/`buyback`/`burn` مسموحة في الوثائق التقنية؛
  ممنوعة في نصوص الواجهة (راجع [`CLAUDE.md §18`](../CLAUDE.md) — الفاعل الذي يحرّك الأموال هو **العقد الذكي**، لا "نحن").
- **القانوني:** مؤجّل أثناء testnet · **إلزامي** (محامي أصول رقمية) قبل الإطلاق العام ([`STATE.md`](../STATE.md), [`PLAN.md §0`](../PLAN.md)).

---

## 9) القرار الاستراتيجي — Validate-First (متى تُطلق العملة؟)

**التوصية المثبّتة في [`STATE.md`](../STATE.md): أجّل `$DSWP` حتى يُثبَت الطلب.**

السبب: العملة = **أكبر تكلفة + أكبر تعرّض قانوني + حاجة سيولة ($250k–$1.25M)** — كل ذلك قبل إثبات GMV.

ترتيب التنفيذ الموصى:
```
MVP (عمولة USDT فقط) ──► إثبات GMV في niche واحد ──► قرار العملة
   ──► عملة + سيولة + تدقيق ──► mainnet
```

أكبر خطر = **cold start** → niche واحد + seeding المنفّذين أولاً، ثم العملة لاحقاً.

---

## 10) العناوين الحيّة والحقائق التقنية

### BSC testnet (97) — مفاتيح ستُدوَّر ([`STATE.md`](../STATE.md))

| العقد | العنوان |
|---|---|
| `$DSWP` | `0x2DD2Cd306f32cd6709d4316EF0df125235654734` |
| Escrow V1 | `0xCEE07220dEC813f8A58b7Da73349dabbc4005840` |
| Escrow V2.1 (الأحدث) | `0x67Eca35d3d23401d53Fba988759F8A649BA67c3e` |
| USDT (mock test) | `0xE950eb93aCa1f29848f5cBac61d78657e3c97287` |

موثّقة على BSCScan testnet (source verified).

### حقائق صلبة (BSC)

| البند | القيمة |
|---|---|
| USDT mainnet (BEP-20) | `0x55d398326f99059fF775485246999027B3197955` — **18 خانة** (لا 6!) |
| PancakeSwap V2 Router | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| Chain IDs | mainnet=56 · testnet=97 |
| Solidity / EVM | `=0.8.24` / `shanghai` |
| OpenZeppelin | v5.1.0 (vendored في `contracts/lib/`) |

---

## 11) المخاطر (تقنية واقتصادية — دون وعود قانونية)

| الخطر | التخفيف |
|---|---|
| فشل السواب يحجب أجر المطوّر | الدفع أولاً + حرق معزول بـtry/catch + تأجيل لـ`buybackReserve` |
| Reentrancy حول الراوتر | CEI صارم + `ReentrancyGuard` + تصفير الرصيد قبل السواب |
| MEV / Slippage | `amountOutMin` + `deadline` + تجميع دوري |
| فشل الحرق في `address(0)` | `ERC20Burnable.burn()` حصراً |
| بركة سيولة ضحلة مبكراً | تأجيل الشراء التلقائي حتى عمق كافٍ + قفل LP |
| إساءة استخدام السكّ | `mint` خلف `Ownable2Step` (multisig على mainnet) + سقف صلب |
| تركّز الملكية | توزيع 4-فئات + vesting الفريق 4 سنوات + قفل LP |
| **بوابة mainnet** | تدقيق مستقل (PeckShield/CertiK) + Multisig 3-of-5 + timelock — قبل أي نشر إنتاجي |

---

## 12) مراجع

- العقد: [`contracts/src/DevSwapToken.sol`](../contracts/src/DevSwapToken.sol)
- التوزيع: [`contracts/script/DistributeToken.s.sol`](../contracts/script/DistributeToken.s.sol)
- الـescrow (الحرق): [`contracts/src/DevSwapEscrow.sol`](../contracts/src/DevSwapEscrow.sol) · [`DevSwapEscrowV2_1.sol`](../contracts/src/DevSwapEscrowV2_1.sol)
- الـkeeper: [`keeper/src/index.ts`](../keeper/src/index.ts)
- العقود (تقني): [`docs/CONTRACTS.md`](CONTRACTS.md) · المعمارية: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) · الأمان: [`docs/SECURITY.md`](SECURITY.md)
- الخطة الاقتصادية: [`PLAN.md §4`](../PLAN.md) · الحالة والقرارات: [`STATE.md`](../STATE.md)
- التشغيل (runbook): [`docs/RUNBOOK.md`](RUNBOOK.md)

> **اتساق الأرقام (CLAUDE.md §17):** أي تعديل لثوابت الرسوم (`FEE_BPS`/`BUYBACK_BPS`) يجب أن يحدّث
> هذه الوثيقة + `web/messages/{en,ar}.json` + `README.md` + `docs/CONTRACTS.md` معاً. الأرقام الحالية:
> **97% مطوّر · 1.5% منصّة · 1.5% buyback-burn · 3% إجمالي**.
