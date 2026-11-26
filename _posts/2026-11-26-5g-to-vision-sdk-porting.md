---
layout: post
title: "從 5G Cloud-Native 到 NI→Hik 視覺 SDK Porting — 一個跨域工程師的落地紀錄"
date: 2026-11-26 14:00 +0800
categories: [career, porting, vision]
tags: [5G, O-RAN, NI-Vision, HikRobot, VisionMaster, CSharp, dotnet, SDK-porting, ADAPT]
author: Pisces Zhang
description: "3.5 年的 5G 雲原生經驗要怎麼轉成 1.2 年的視覺 SDK porting?這篇寫架構選型、訊息相容性設計、缺陷定位的踩坑經驗,給面對類似 porting 工作的工程師。"
---

## 前言 — 為什麼要寫這篇

我在 PEGATRON 待了 4+ 年。前 3.5 年做 5G Cloud-Native(O-RAN Hierarchical M-Plane、CBRS 北美認證、GitLab CI/CD 容器化部署),1.2 年前內部轉到自動化團隊,接手 NI Vision → HikRobot VisionMaster 的 SDK porting 專案。

很多人聽到「跨域」會問:「會不會覺得降級?」「從 5G 跳去做視覺,薪資有差嗎?」這篇不回答那些問題,只回答一個:**跨域 porting 的工程方法論是什麼,以及我從雲原生出身做視覺 SDK porting 一年多,踩過哪些坑、修了哪些 bug、學到了什麼**。

如果你是正在評估要不要換領域的工程師、或手上有一個「舊 SDK 換新 SDK」的 porting 任務,這篇應該能幫上忙。

---

## 1. 跨域的本質:不是換技術,是換方法論

從 5G Cloud-Native 跨到視覺 SDK porting,技術棧變了(從 Python + Docker/K8S/GitLab CI/CD,變成 C# + .NET 4.7.2 + WinForms + Hik MVD SDK V4.4.3)。但**工程方法論沒變**:

- **API boundary contract** — 5G 那邊用 Netconf/YANG 跟基站通訊,視覺這邊用 SDK 跟相機/影像源通訊;契約面對的失敗模式是一樣的(版本差異、time-out、欄位缺失)
- **byte-identical 相容性** — 5G 那邊跟 SAS 伺服器介接要符合 CBRS 認證(任何 byte 變動就失去認證),視覺這邊訊息格式 `[name],Xxx:T/F,...` 是下游流程依賴的契約;**任何字元變動就斷線**
- **CI gate 守住編譯** — 5G 那邊靠 GitLab CI/CD 把所有單元測試跑完,視覺這邊因為 msbuild 在我們環境不能用,我改用 Roslyn csc 寫了一個 `build_hik.ps1`,每個 commit 都跑 0 errors 才合進去
- **ADAPT 標註** — 對 SDK 行為沒把握時,留 `// ADAPT: 待確認` 標記,不要 ship 猜測的程式碼

這四條**就是我從雲原生帶過來的東西,在視覺 SDK porting 一樣成立**。所以跨域不是降級,是同一套工程紀律套到不同領域。

---

## 2. 架構對齊:不是重寫,是對齊

接手的第一個任務,是要把 NI Vision 的 8 個演算法移植到 HikRobot VisionMaster SDK。NI 是 LabVIEW 體系的 SDK,Hik 是 C# 的 MVD SDK(V4.4.3)—— 兩邊的 API 風格、套件分佈、生命週期設計都不同。

平台本身已經有既定的 10-file 結構、reflection registry、INI 設定慣例、byte-identical 訊息格式。**porting 的策略不是「重寫」而是「對齊」**:

- 演算法類別繼承 `Algorithm` 基底,只 override `Process(image)` 跟 `ReadSettingFile()`
- `this.type` 用全域名(如 `"HikRobotCV.MultiGrayPatMatch"`),這是 registry、INI、session 三方共用的 key
- 訊息透過 `msg_container.msg` 寫入,格式嚴格遵守 base 的 `[name],Xxx:T/F,...` 慣例
- 共用的 overlay(綠色 ROI + 紅色結果)透過 `HikImgOverlay` 處理

這套架構一對齊,**8 個演算法就可以用「填 10 檔模板」的方式快速產出**。整個 porting 階段的 commit 是 cfcb789(23 檔,+4095/−15)。

---

## 3. 訊息相容性設計:byte-identical 是契約,不是口號

下游流程對我們的訊息有兩個依賴:

1. **契約式解析** — 用 `[name],Xxx:T/F` 切欄位,任何字元變動就 parse 失敗
2. **歷史 log 相容性** — 既有 ExceptionLog 是用 NI 的格式寫的,如果 porting 後訊息差一個空白,後續比對工具全斷

所以 porting 的紀律是:**寫出去的訊息必須 byte-identical 跟 NI 相同,包括逗號、空白、key 名稱**。

舉例:`PatternMatch:T,MatchNumber:1,Point_X:1119.000,Point_Y:1630.000,Angle:0.000` —— 光是 `Point_X:` 後面有沒有小數點、`MatchNumber:1` 還是 `MatchNumber: 1`,都是契約的一部分。

實作上有兩個具體設計:

- **Process gate 一致性** — 不論底層 SDK 怎麼回傳,Process 的出口必須走相同的 msg 字串生成路徑
- **Test fixture 用 NI baseline** — 把所有 NI 生成的訊息 dump 出來當 fixture,porting 後對 fixture 做 byte-level diff

我在 porting 過程中犯過一個錯誤:把 `Point_X:1119.000` 改成 `Point_X: 1119.000`(冒號後多加空白),fixture test 立刻 fail,我在 commit 前修正。**沒有 fixture 就沒有 byte-identical**。

---

## 4. 缺陷定位紀錄:4 個我修過的真實 bug

這部分是寫給 porting 工程師看的——**NI → Hik porting 的常見陷阱**。

### Bug 1:SDK 參數 key 被 try/catch 靜默吞掉

```csharp
// 原始(錯誤):用 NI 參數名直接搬到 Hik SDK
ApplyRunParam("EdgeWidth", width);
ApplyRunParam("FindMode", mode);
ApplyRunParam("CaliperNum", count);

// Hik SDK 對不認得的 key 不丟例外(被 try/catch 吞掉)
// 工具一直跑在預設值(30 條 caliper、5px 寬、Best 模式)
```

正解 — 對 CHM 一個 key 一個 key 驗。`EdgeWidth` 實際叫 `RegionWidth`、`FindMode` 叫 `LineFindMode`、`CaliperNum` 叫 `RayNum`。

**教訓**:**任何 SDK 的 run param key 都要對照官方文件逐一驗證**。SDK 不會跟你說「這個 key 我不認得」——它只是默默忽略你。

### Bug 2:caliper 角度符號反了

```csharp
// 原始(錯誤):用 ROI rect 的 center + cos/sin(Angle)
double rad = rrect.Angle * Math.PI / 180.0;
var start = new MVD_POINT_F(
    (float)(rrect.Center.X - half * Math.Cos(rad)),
    (float)(rrect.Center.Y - half * Math.Sin(rad)));

// ROI 畫面是綠色的,但 caliper 引導線沿「鏡像 2×Angle」方向搜尋
// 軸向 ROI 看不出來;旋轉 ROI 就明顯錯邊
```

正解 — 直接用 ROI rect 自己的 `P_Left` / `P_Right`(已經處理過平台慣用的 `-Angle` 慣例),不要再算一次。

**教訓**:**幾何自洽**比數學正確更重要——同一個 ROI 的「畫面」、「搜尋」、「結果」三個座標系必須一致。

### Bug 3:FindOrient 漏送

INI 的 `RakeDirection`(搜尋方向)對應的 SDK 參數叫 `FindOrient`,但我在 `ApplyRunParam` 完全沒送——所以 INI 設得再怎樣,搜尋軸永遠是預設值。

**教訓**:**所有 INI 欄位都必須有對應的 SDK 送出邏輯**,沒送就等於沒設。

### Bug 4:port 時 `|| setting.ExactMatch` 掉了

NiCV 原式是 `result.Count != requested || setting.ExactMatch`,我 port 到 Hik 時把後面那段掉了——意思是 ExactMatch 開關永遠失效。

**教訓**:NiCV 程式碼要先讀完,**port 不是「看起來對就對」**,是要**逐行比對語意差異**。

---

## 5. 工具與紀律:ADAPT 原則 + Roslyn CI gate

兩條紀律支撐整個 porting 過程。

**ADAPT 原則**:對 SDK 行為不確定時,寫 `// ADAPT: 待確認` 標記,**不 ship 猜測的程式碼**。在 commit 訊息、code review、文件裡都標出來,後續實機驗證時一個一個清除。

```csharp
// ADAPT: 待確認 — FindOrient 值形式用 enum 名,若 SDK 不認則換數值
ApplyRunParam("FindOrient", MapFindOrient(setting.RakeDirection));
```

**Roslyn-based CI gate**:msbuild 在我們環境因為 NuGet 缺包跑不起來。我寫了一個 `build_hik.ps1`,用 Roslyn csc 直接單編譯整個 HikRobotCV,每個 commit 都跑——`csc exit 0` 是 commit 前的唯一門檻。

```powershell
powershell -ExecutionPolicy Bypass -File build_hik.ps1
# 失敗就不 push
```

**好處**:**繞過了環境問題、把編譯錯誤從 PR review 階段提前到 commit 階段**。本來 PR review 才會被 build error 擋下來,現在 local commit 就先擋——少一次 round-trip,也少一次「為什麼 main branch 是壞的」。

---

## 6. 結語:跨域工程師的價值在哪

一年多做下來,我發現跨域工程師的價值**不在於「會的語言多」**,而在於**「能在兩個領域之間看見同一套方法論」**。

5G 那邊用 pytest + CI/CD 把測試流程自動化,搬到視覺 porting 就是「每個演算法 commit 前都跑 unit test」;5G 那邊用 docker 把環境隔離,搬到 Roslyn-based build 就是「把 build 環境從 Visual Studio 解耦」。這些不是巧合,是同一套工程哲學。

如果你正面對類似的 porting 工作(換 SDK、換框架、換語言),我的建議是:

1. **不要重寫,要對齊**——找出既有架構的最小不可變集合,新 SDK 套進去
2. **byte-identical 是契約,不是口號**——下游流程依賴的東西一個字都不能動;用 fixture test 守住它
3. **每個 SDK 參數 key 對 CHM 一一驗**——`try/catch` 不會保護你
4. **ADAPT + CI gate**——給「未驗證」留出口,把「已知壞」擋在 local commit 前

跨域不浪漫,但**值得**。

---

## 附錄:本系列涉及的工程紀錄

- 開發分支:`feat/hikvm-multi-patmatch`(已合併 main)
- 主要 commit:`cfcb789`(23 檔,+4095/−15,Multi-Gray + Multi-Contour + L1/PM 修正)
- 合併至 main:`720249d`
- L1 修的 3 類 bug:SDK key 對齊、caliper 角度符號、FindOrient 漏送
- PM 修的 1 個 bug:`|| setting.ExactMatch` 補回

> 註:本系列程式碼涉及公司內部資產,僅以工程方法論公開分享;具體 commit hash 與 repo 結構作為個人開發紀錄保留。

---

**作者**:Pisces Zhang — Software Engineer @ PEGATRON,4+ 年跨 5G Cloud-Native 與工業視覺 SDK 整合經驗。 [LinkedIn](https://www.linkedin.com/in/hong-rui-zhang-1a248016b/) · [GitHub](https://github.com/rexlim820220)
