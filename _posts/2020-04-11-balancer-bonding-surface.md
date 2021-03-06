---
layout: post
title:  "Balancer's Bonding Surface — 自動再平衡指數基金"
date:   2020-04-11 00:00:00 -0500
categories: crypto
---

本文的目的在於介紹 Balancer 的 bonding surface ([白皮書](https://balancer.finance/whitepaper/))，可以用來實現自動指數基金，不需要雇用人類 fund manager 負責 portfolio rebalance。

本文思路是：先思考 Uniswap，回顧 Uniswap 的起源和思路，再探討 Balancer bonding surface 的原理。

從理論、實務，再到獲取早期用戶的信任，一小口一小口地，去中心化金融正在侵蝕傳統金融板塊。

### Uniswap 的起源
以第一性原理來思考：在以太坊上建立交易所，會遇到什麼問題？

我們得先回答：什麼是交易所？

傳統交易所的模式是，給交易者上來掛買單(bid) 和賣單(ask)，交易所依照其所遵循的各種原則實作並運行「交易撮合演算法」，撮合買家掛的買單和賣家掛的賣單。

在流動性高，也就是很多人交易、很多錢換手的交易所裡，任何時候都有非常高頻次且大量的掛單和撤單，因為在這個程式化交易的時代，交易程式們會剖析掛單簿 (order book) 和委託單流量 (order flow)，輔以交易演算法進行自動交易。掛單頻率以做市商 (market maker) 的交易程式為最，左手掛賣單和買家成交，右手掛買單和賣家成交。

做市商的盈利模式就是在左手賣右手買之間，賺取買賣價差 (bid-ask spread)。做市商的有效運行，能使交易所維持高流動性、買賣價差較小。所以做市商又名流動性提供者。

在以太坊上建立這樣交易撮合式的交易所，問題就在於以太坊上的交易 — 只要是在以太坊虛擬機上執行操作 — 都要付 gas。在不考慮擴容方案 (例如狀態通道) 的前提下，掛單和撤單都會消耗 gas。這讓做市的操作非常昂貴，導致缺乏做市商的存在，使早期去中心化交易所上的買賣價差都十分大，並且小小的交易量就會推動價格有感變化。

那麼，能不能想出一種不進行買賣撮合的交易所模式，適合在以太坊上運行？

### Uniswap 的作法，以及所謂 bonding curve
歡迎 Uniswap 登場。Uniswap 是以太坊上的一個自動做市合約。在 Uniswap 的世界裡，沒有掛單簿，也沒有撮合算法。

再從第一性原理來思考：做市商的目標是什麼？

舉個例子([source](https://blog.gnosis.pm/building-a-decentralized-exchange-in-ethereum-eea4e7452d6e))，今天你想要在冰淇淋市場上當一名自動做市商。你左手賣冰淇淋買進美分(penny)，右手賣美分買進冰淇淋。作為一名想要永續經營的做市商，你的動機非常簡單：手上隨時有足夠的冰淇淋和美分來應付任何可能發生的市場狀況。什麼狀況呢？往極端想：一，如果今天天氣爆炸熱，一堆人跑來找你買冰淇淋，導致你的冰淇淋存貨下降；二，今天天氣爆炸冷，一堆人跑來找你賣對他們已毫無吸引力的冰淇淋換成美分，導致你的美分存貨下降。只要你的冰淇淋存貨或是美分存貨降到零，你就得被迫停止做市，去其他市場上補貨。想當自動做市商的你最討厭這種麻煩的情形了。

因此，必須存在某種機制讓你的存貨不會無止盡下降。比較粗暴的做法是，當某存貨降到一定水準以下，你就暫停該方向的交易。例如冰淇淋只剩20%存貨時你就不賣冰淇淋了，直到天氣轉冷，有人找你賣冰淇淋為止。比較優雅、連續的作法是，你左手掛出的冰淇淋賣價隨著你的存貨水準動態調整，存貨越少，賣價越貴。如果你手上只剩趨近於0的冰淇淋，則理論上你掛出的冰淇淋賣價應該要趨近於無窮大，保證勸退所有嘴饞的人。這樣做，讓你理論上不可能耗盡任何一種存貨，對吧？

好，那這個動態調整的價格如何實現？一個簡單的做法就是：左手掛出的冰淇淋賣價 = 美分存貨÷冰淇淋存貨。只要你還有足夠的美分存貨，則趨近於0的冰淇淋存貨將導致賣價飆升到天文數字。右手則是冰淇淋買價 = 冰淇淋存貨÷美分存貨。

下式描述冰淇淋賣價，*p* 是價格，*q* 是存貨，*P* 是美分，*I* 是冰淇淋：

<div align="center">
<img src="/assets/eq_uniswap_1.svg"/>
</div>

這個賣價的意思是，如果在現有存貨的狀態下，有人向你購買無窮小份的冰淇淋，就該用這個價格跟他交易。

往源頭走，我們想起雙曲線 xy=k 上的任一點斜率都是 -y/x。而在冰淇淋存貨對美分存貨作圖，函數的斜率正是交易價格的意思 – 兩存貨微幅變動的比值不就是價格嗎！因此上述的數學關係其實可以從一個恆定式推導而出：

<div align="center">
<img src="/assets/eq_uniswap_2.svg"/>
</div>

上式表示，任何時候你手上的冰淇淋存貨乘以美分存貨都是某定值 (invariant)。你常聽到的 *bonding curve* 就是在描述上述這條雙曲線。它打破了傳統觀念裡，市場價格必定由買賣雙方一起決定的說法。在上述模型裡，價格在任何時候都依照賣方存貨比值而定。

舉例：今天有個買家跟你說，他要買50美分(下式中的 *p*)的冰淇淋，請問能買到多少冰淇淋呢(下式中的 *i*)？你看了看你當初開始營運時的起始存貨：100份冰淇淋與100美分。使用下式計算：

<div align="center">
<img src="/assets/eq_uniswap_3.svg"/>
</div>

你就能算出這個買家的50美分能換得多少冰淇淋了：

<div align="center">
<img src="/assets/eq_uniswap_4.svg"/>
</div>

這一對存貨(冰淇淋和美分)的總和就是所謂的流動池(liquidity pool)。任何一種 ERC-20 代幣和 Eth 兌換的流動池都能在 Uniswap 上找到。你也可以在 Uniswap 上為自己發佈的 ERC-20 代幣創建一個流動池。

### Balancer 的 bonding surface

Balancer 想做的事，就是把上述原理概括到「雜貨市場」的情形。假如今天，你想同時在冰淇淋、檸檬汁、香菸和口香糖市場裡當做市商，能否設計一種機制，讓你成為一名自動做市商、支援任何一對商品的兌換？ (e.g. 我可以拿口香糖跟你換香菸、拿檸檬汁跟你換冰淇淋)

Balancer 辦到了。定義以下價值函數：

<div align="center">
<img src="/assets/eq_balancer_v.svg"/>
</div>

其中，t 指代任何一種代幣，B_t 是代幣 t 的存貨價值，W_t 是代幣 t 佔流動池總價值的權重，應保持恆定，由做市商自行定義。有了這個式子我們就能算出做市商賣出 o 幣、買進 i 幣的兩幣匯率/現貨價格 (spot price)：

<div align="center">
<img src="/assets/eq_balancer_sp.svg"/>
</div>


這是 Uniswap bonding curve 往高維度的推廣，因此稱為 bonding surface。

### 不需要基金管理人的指數基金

Balancer 的 bonding surface 實現兩個目標：
1. 自動再平衡 (rebalance) 的投資組合 — 不收取管理費的自動指數基金。
2. 成為價格信號提供者。

讓我們再以第一性原理理一次思路。

指數基金的目標很簡單：維持各部位之間相對價值恆定。

設想以下情境：今天，一個以太坊的地址 X 想要保持持有 A 幣與 B 幣價值 7:3 的權重，同時，地址再平衡前後總價值不變。

- 時刻 1：地址 X 持有 50 顆 A (幣價 $100) 和 100 顆 B (幣價 $50)。
  - 兩幣總價值為 1:1。
- 時刻 2：A 幣幣價下跌至 $80，B 幣幣價下跌至 $30。地址 X 需要做再平衡。
  - 50x$80 = $4000、100x$30 = $3000 => 基金總價值 $7000。為了保持權重 1:1，兩幣價值應同為 $3500。
- 時刻 2+：再平衡交易發生。
  - 6.25 顆 A 幣價值 $500 變賣成 16.66667 顆 B 幣。
  - A 幣從 50 顆變成 43.75 顆；B 幣從 100 顆變成 116.66667 顆。
  - 基金持有：43.75x A、116.667x B。兩幣價值皆為 $3500。

關鍵在於：再平衡的交易 (將 6.25xA 交易成 16.6667xB) 由誰來完成？
- **市面上的指數基金**：基金管理人，收取管理費，進行交易、記帳、稅務管理等。
- **Balancer 指數基金**：外部游擊交易者來完成交易，獲得套利收益。

如果，需要再平衡發生的時候，自動出現套利機會，就能激勵任何人來幫基金完成再平衡了！

但是，為什麼會有套利機會？

因為任何人在任何時候都能和 Balancer 指數基金買賣，基金依照 bonding surface 對外公布的動態匯率進行交易。

回顧上式 spot price：基金向外公布的 A-B 交易對匯率是：

<div align="center">
<img src="/assets/eq_balancer_ab.svg"/>
</div>

因此：
- 時刻 1，基金 A-B 匯率是 (50/1) / (100/1) = 1:2，等同基金外的 A-B 匯率
  - 外界 A 幣幣價 $100 而 B 幣幣價 $50。
- 時刻 2，基金 A-B 匯率依舊是 1:2，但外界 A-B 匯率已變成 1:2.6667，
導致套利機會的出現，使外部任何交易者有誘因與基金交易，藉此獲利。
  - 以 B 計價，基金收 2B 給 A，但外界已經是收 2.6667B 給 A。
  - 等於是基金提供的 A 比較便宜。
  - 因此套利者有意願提供 B 向基金便宜買到 A，拿去外界變賣。
  - 套利過程中，基金 A 存貨變少，B 存貨變多，依照 bonding surface 的規則，A-B匯率上升。
  - 隨著基金被套利，基金內 A 與 B 的存貨持續變化，直到基金的 A-B 匯率與外界相同，套利機會消失，同時再平衡也完成了。

#### 我的想法：
- Balancer 實現激勵機制，讓一個地址的各幣價值比自動維持恆定。基金提供套利機會，讓外界套利者進來幫基金完成再平衡。
- 如果今天，這個地址是一個國家呢？呼應中國央行研究報告「以人的價值為基礎的數字貨幣」而有以下思路：
  - 國家持有所有國民一定數量的個人通證 (token)。
  - 任何個人通證皆反映該國民自己的經濟價值，是綜合了該國民所有收入、支出、投資、儲蓄的指標。(參考 Ryan Adams 的 [How to tokenize yourself](https://bankless.substack.com/p/how-to-tokenize-yourself-full))
  - 任何國民的經濟價值上升，都會自動反應在國家持有的總價值和 GDP 上升。
  - 任何國民的經濟價值下滑，都會自動反應在國家持有的總價值和 GDP 下滑。
  - 各國民通證在國家持有裡的權重若能強制保持在為 1:1:1....:1，則實現了
    - 國家價值對任何國民經濟價值變動的敏感度相同。
    - 而國家的施政直接由此價值和 GDP 動態制定和執行。
    - 因此，這個模型或能實現國家自動地平等重視所有國民的經濟條件。
- 國家這個詞或許已乘載太多含義和聯想，不適合我們的未來。上述這個模型可以套用在任何基於區塊鏈個人通證所實現的人類組織，可以適用於很未來主義的、所有民族意識主權意識消弭後的全球集體意識，由此模型自動統籌資源，真正實現無國界的民有、民治、民享。或許隨著這一天的到來，人類才有足夠的協同動能一起往宇宙發展吧。
