---
marp: true
paginate: true
---

# Bitcoinのトランザクションを読む

---

## なぜHEX表記のトランザクションを読むのか

- デバッグ
  - エクスプローラーで確認できるのはブロードキャストに成功したものだけ
  - 高機能なエクスプローラーやツールがないチェーンもある
  - エクスプローラーやツールが隠してしまう情報もある
- ライブラリの作成や理解
  - トランザクションの処理に便利なライブラリがない環境もある
  - ライブラリを検証して使うには理解する必要がある
- 「送金する」とは何をしているのかという理解が深まる

---

## 例に使うトランザクション

https://btc.com/btc/transaction/b47513763d97214c4bbdaa29c886741099b3737023a480a455f74b9e53a2728e

`01000000016291cb3e72f9daf6b8f713bcc8add9e6f349eae56df7b5e987ea00d7cf106dd1000000006b483045022100dda999d60951db26111097442d479b3ae0631f5a25591d52483b61639d96d39802205340a83611316f7aa9899dbbc1f74e731742c63fa93296927fceb4498ae78ee3012102f801663fba078188413f43e2e492e705717e73049d9a534c19998d6b983fdf70ffffffff02384a0000000000001976a914eea60aa94840a90cb7970abb2a1b2cea7aa9b5e488ac7cda04000000000017a914ecb6fa6849118eaf5de71c0f4037d0e50a98db988700000000`

---

## トランザクションの構造

```
バージョン: 32bit整数
inputの数: 可変長整数
inputの配列:
  txid: 32バイト
  index: 32bit整数
  scriptSig: 可変長
  sequence: 32bit整数
outputの数: 可変長整数
outputの配列:
  送金額: 64bit整数
  scriptPubKey: 可変長整数
lockTime: 32bit整数
```

参考: https://en.bitcoin.it/wiki/Transaction

---

### 可変長整数

- 0x00 - 0xfd\
  そのまま1バイト
- 0xfe - 0xffff\
  0xfd + 2バイト = ３バイト
- 0x1_0000 - 0xffff_ffff\
  0xfe + 4バイト = ５バイト
- 0x1_0000_0000 - 0xffff_ffff_ffff_ffff\
  0xff + 8バイト = ９バイト

---

<style>
.version {
  color: black;
}
.var-int {
  color: red;
}
.txid {
  color: blue;
}
.int {
  color: teal;
}
.script {
  color: green;
}
</style>

## トランザクションを読んでみる

<code><span class="version">01000000</span><span class="var-int">01</span><span class="txid">6291cb3e72f9daf6b8f713bcc8add9e6f349eae56df7b5e987ea00d7cf106dd1</span><span class="int">00000000</span><span class="var-int">6b</span><span class="script">483045022100dda999d60951db26111097442d479b3ae0631f5a25591d52483b61639d96d39802205340a83611316f7aa9899dbbc1f74e731742c63fa93296927fceb4498ae78ee3012102f801663fba078188413f43e2e492e705717e73049d9a534c19998d6b983fdf70</span><span
class="int">ffffffff</span><span class="var-int">02</span><span class="int">384a000000000000</span><span class="var-int">19</span><span class="script">76a914eea60aa94840a90cb7970abb2a1b2cea7aa9b5e488ac</span><span class="int">7cda040000000000</span><span class="var-int">17</span><span class="script">a914ecb6fa6849118eaf5de71c0f4037d0e50a98db9887</span><span class="int">00000000</span></code>

- sequenceはたいてい`0xffffffff`
- 送金額はたいてい後半の`0x00`が並んでる辺り
- scriptPubKeyのサイズが`0x19`ならほぼP2PKH宛
- scriptPubKeyのサイズが`0x17`ならほぼP2SH宛

---

<style>
.push-data {
  color: red;
}
.signature {
  color: green;
}
.pubkey {
  color: blue;
}
</style>

## スクリプトを読んでみる

- OP_CODEの仕様どおりに分解する

### P2PKHのscriptSig

- 署名と公開鍵のデータだけが入っている

<code><span class="push-data">48</span><span class="signature">3045022100dda999d60951db26111097442d479b3ae0631f5a25591d52483b61639d96d39802205340a83611316f7aa9899dbbc1f74e731742c63fa93296927fceb4498ae78ee301</span><span class="push-data">21</span><span class="pubkey">02f801663fba078188413f43e2e492e705717e73049d9a534c19998d6b983fdf70</span></code>

---

<style>
.sighash {
  color: blue;
}
.identifier {
  color: red;
}
.length {
  color: teal;
}
.value {
  color: green;
}
</style>

#### 署名を読んでみる

- 署名本体とsighashで構成されている
- ECDSAの署名本体はASN.1のDERエンコードで表現されている
  - Schnorr署名は別

<code><span class="identifier">30</span><span class="length">45</span><span class="identifier">02</span><span class="length">21</span><span class="value">00dda999d60951db26111097442d479b3ae0631f5a25591d52483b61639d96d398</span><span class="identifier">02</span><span class="length">20</span><span class="value">5340a83611316f7aa9899dbbc1f74e731742c63fa93296927fceb4498ae78ee3</span><span class="sighash">01</span></code>

- ASN.1は`<Identifier><length><value>`の構造
  - 登場するIdentifier
    - `0x30`: Sequence、配列のようなもの
    - `0x02`: Integer、Big Endianの整数値
- rとsが順番に並んでいる

参考: http://shaw.la.coocan.jp/security/asn1/ber.php （文字化けする）

---

<style>
.dup {
  color: blue;
}
.hash160 {
  color: orange;
}
.hash {
  color: green;
}
.checksig {
  color: teal;
}
.equalverify {
  color: deeppink;
}
.equal {
  color: deeppink;
}
</style>

### P2PKHのscriptPubKey

<code><span class="dup">76</span><span class="hash160">a9</span><span class="push-data">14</span><span class="hash">eea60aa94840a90cb7970abb2a1b2cea7aa9b5e4</span><span class="equalverify">88</span><span class="checksig">ac</span></code>

`<DUP> <HASH160> <public key hash> <EQUALVERIFY> <CHECKSIG>`

### P2SHのscriptPubKey

<code><span class="hash160">a9</span><span class="push-data">14</span><span class="hash">ecb6fa6849118eaf5de71c0f4037d0e50a98db98</span><span class="equal">87</span></code>

`<HASH160> <script hash> <EQUAL>`

- 「アドレス」は緑の部分を人間に読みやすくしたもの

---

<style>
.marker {
  color: purple;
}
.flag {
  color: fuchsia;
}
.witness {
  color: olive;
}
</style>

## Segwit編

- versionのあとにmarkerとflagがある（今のところ0x0001固定）
- scriptSigは0バイト
- outputのあとにWitnessがある

---

## 例に使うトランザクション

https://btc.com/btc/transaction/f99975990dd9b6c52122f5f43edd200c2f50840998a31ecf061617dbf461d150
<code><span class="version">01000000</span><span class="marker">00</span><span class="flag">01</span><span class="var-int">02</span><span class="txid">033f30299ca503c8fede66d5243a8a457a496a1665d61e09e2ccbcbb52fed63c</span><span class="int">00000000</span><span class="var-int">00</span><span class="int">ffffffff</span><span class="txid">2796aea915dfcf2536fc13be81dca1e6c356138b7bc1b43d51734293d26004f3</span><span class="int">00000000</span><span class="var-int">6b</span><span class="script">48304502210083b87bab737bc853bba75ef2c21fb5181756bb1b5feb898c79521482c21756d70220702c48227db3e9e43bb93ec737774fd3e3a828363bf95a917d9607bde9e59e0901210224fc256a350a44f6a9d0b7709ff39329bfa88000e3db9970c4986f6d48713367</span><span class="int">ffffffff</span><span class="var-int">02</span><span class="int">048d000000000000</span><span class="var-int">16</span><span class="script">0014ced2542a63b6b380a8865c19a3e4643811e854a2</span><span class="int">6a27000000000000</span><span class="var-int">17</span><span class="script">a91499cc698a705d0663e2f0444587c8c902b8009d9987</span><span class="witness">02483045022100967f1a8b20ccd969aeb07c49ddd98a8ef901807238baf55291e67cce73d395c6022018b709c029e27305884f47b62ba0445d460cac8b4fb1ee20f9efdd1522aef87601210225d02c47999448f9898d08c946fed714919f6ef88df4373404a056c5bd96f53c00</span><span class="int">00000000</code>

---

### Witnessを読んでみる

- inputと同じ順番で、対応するWitnessデータが並んでいる
- `<number of stack items> <stack item>...`という構造
- scriptではない

<code><span class="var-int">02</span><span class="var-int">48</span><span class="signature">3045022100967f1a8b20ccd969aeb07c49ddd98a8ef901807238baf55291e67cce73d395c6022018b709c029e27305884f47b62ba0445d460cac8b4fb1ee20f9efdd1522aef87601</span><span class="var-int">21</span><span class="pubkey">0225d02c47999448f9898d08c946fed714919f6ef88df4373404a056c5bd96f53c</span><span class="var-int">00</code>

参考: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki

### scriptPubKeyを読んでみる

<code><span class="version">00</span><span class="var-int">14</span><span class="hash">ced2542a63b6b380a8865c19a3e4643811e854a2</span></code>

`<0> <public key hash>`

---

## 終わりに

無味乾燥な16進数の羅列が意味のある構造に見えてくることを願います
