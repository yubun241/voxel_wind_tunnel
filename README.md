# ボクセル風洞 (Voxel Wind Tunnel)

STL を読み込み、ブラウザ内の WebGPU で格子ボルツマン法 (D3Q19 + Smagorinsky LES) を回し、
流れの可視化と Cd / Cl / ダウンフォースの A/B 比較を行う簡易風洞です。自動車部品の形状比較を想定しています。

## データの扱い

- **サーバーには何も送信しません。** GitHub Pages は `index.html` を配信するだけで、STL の読込・ボクセル化・計算・描画はすべて閲覧者のブラウザ内で完結します。
- **何も保存しません。** STL も比較結果もタブのメモリ内にだけ存在し、localStorage / sessionStorage / IndexedDB / Cookie は一切使いません。タブを閉じる（または再読込する）と消えます。「すべて消去」ボタンで即時に破棄もできます。
- 唯一の外部通信は Three.js (描画ライブラリ) を jsDelivr CDN から読み込む 1 回だけです。完全オフライン化したい場合は後述。

## 動作要件

| 項目 | 内容 |
|---|---|
| ブラウザ | WebGPU 対応: Chrome / Edge 113+、Safari 26+ (macOS/iOS)、Firefox 141+ (Windows のみ)。 |
| 接続 | HTTPS (WebGPU は secure context 必須。GitHub Pages は標準で HTTPS)。 |
| GPU | 統合 GPU でも動作。160×80×80 格子で約 100〜300 MB の VRAM。256×128×128 は 1 GB 前後。 |

計算は**閲覧者の GPU** で走ります。ホスティング先 (GitHub Pages) の CPU/GPU は使われないため、
Claude のアーティファクトとして動かしていた時と性能は変わりません。

## デプロイ手順 (GitHub Pages, ブランチ配信)

```bash
git init
git add .
git commit -m "Voxel wind tunnel"
git branch -M main
git remote add origin git@github.com:<user>/<repo>.git
git push -u origin main
```

GitHub のリポジトリ → **Settings → Pages → Build and deployment**:
- Source: **Deploy from a branch**
- Branch: **main** / **/ (root)** → Save

数分後に `https://<user>.github.io/<repo>/` で公開されます。`.nojekyll` は Jekyll 処理を止めるためのファイルです (削除しないでください)。

## ローカルで試す

`file://` では ES Modules の読み込みが拒否されるので、簡易サーバーで開きます。

```bash
python3 -m http.server 8000
# → http://localhost:8000/  (localhost は secure context 扱いなので WebGPU が動く)
```

## 完全オフライン化 (任意)

Three.js を同梱すれば外部通信ゼロになります。

```bash
mkdir -p vendor
curl -L -o vendor/three.module.js https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js
```

`index.html` の import を `./vendor/three.module.js` に書き換えてください。

## 使い方の要点

1. **モデル**: STL (バイナリ / ASCII) を読み込むか、検証用プリセット (球 / 立方体 / Ahmed body) を選ぶ。単位・流れ方向・上方向を CAD 座標に合わせる。
2. **向き**: ヨー / ピッチ (迎角) / ロールのスライダ、または **Alt + ドラッグ** で回す。変更は即座にプレビューされる。
3. **境界・取付**: 車体に付く部品は「取付面」で底面 / 上面 / 側面を壁に密着させる。壁は「静止壁」(車体表面) か「滑り壁」(対称面・半モデル) を選べる。密着面は流体に触れないので力に寄与しない。
4. **準備 (ボクセル化)** → **実行**。右パネルの「流過回数」が 2 を超えてから平均値を読む。
5. **保存** で比較表に登録し、形状を変えた STL で同じ格子・同じ U のまま再実行して差を見る。

### 取付面の考え方

ウイングのエンドプレート、ディフューザー、カナードなどは単体で空中に浮かべると、取付側にも流れが回り込んで実車と違う結果になります。
取付面を壁に密着させると、その面が計算領域の壁 (第 1 層) に埋め込まれ、露出面だけに流体が当たります。
壁は無限平板なので車体の曲率は再現しませんが、「取付側の回り込みがない」という条件は実車と一致します。

## 数値の信頼性について

- ブラウザで扱える格子 (〜256³) では境界層を解像できないため、Cd の**絶対値**は実験値から数十 % ずれ得ます。
  右パネルの「Re (格子で解像)」が物理 Re より数桁低いのがその表れです。
- 同じ格子・同じ条件での**形状 A/B の差**は傾向として信用できます。角で剥離が決まる形状 (ディフューザー、ウイング端板、バンパー) ほど相性が良く、丸い形状 (剥離点が Re 依存) ほど注意が必要です。
- まずプリセットの Ahmed body (Cd ≈ 0.285) で、使う格子での誤差感を掴んでから部品に進んでください。

## 構成

```
index.html   # 単一ファイル。UI / STL パーサ / ボクセル化 / WGSL カーネル / Three.js 描画
.nojekyll    # GitHub Pages の Jekyll 処理を無効化
```

ソルバは `GPUSolver` クラス (`init / setParams / step / sampleForce / sampleField`) に閉じています。
同じ interface で OpenFOAM 等のリモートバックエンドを差し込める構造です。
