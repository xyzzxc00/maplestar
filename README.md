# ⚔ 楓星升等計算器 (MapleStar Leveling Calculator)

楓之谷練等經驗計算工具，純前端單頁應用（HTML/CSS/JS），資料庫使用 Firebase Firestore。

線上使用：https://xyzzxc00.github.io/maplestar/

## 功能

- 輸入等級範圍、目前進度、經驗速率、加倍倍率，算出升等所需經驗與時間
- 皇家騎士團模式（升等自動補 10% EXP，上限 120 等）
- 多段練等計算（一次規劃多個等級區間）
- 加倍卷排程試算（每天打幾小時、手上持有幾張）
- 倒數計時器 + EXP 測速（實測經驗速率）
- 社群經驗資料庫（玩家分享各職業/地圖的實測經驗速率，Firestore 儲存）
- 深色 / 淺色主題切換，支援手機版

## 開發

單一靜態檔案 `index.html`，不需要建置流程，本機用任一靜態伺服器打開即可，例如：

```bash
npx serve .
```

## Firebase

Firestore 規則設定在 [firestore.rules](firestore.rules)，用以下指令部署：

```bash
firebase deploy --only firestore:rules
```
