# SelectBuilder 适配最新 DSP 的笔记

当前 `SelectedBuild.cs` 用的是 DSP 早期 Unity 2018 时代的 API，已经不能编过最新版 `Assembly-CSharp.dll`。要想恢复使用，需要改两块。

## 1. 绘制矩形选框（`GIRer` 整个类被删）

`DrawFrame2D` / `DrawRect2D` 原本是 DSP 自己包的 2D 立即模式绘制 API。替代方案：

- **最简单**：`UnityEngine.UI.Image` + 一块透明面板，`Update` 里算 rect 改 `RectTransform.sizeDelta / anchoredPosition`，不用自己画。
- **一次性**：`OnGUI` + `GUI.DrawTexture`（代码最少，效率稍低但够用）。
- **最贴近原行为**：`GL.Begin(GL.LINES)` 自己画线框（需要挂一个带 `OnPostRender` 的相机组件）。

## 2. `PlayerAction_Build` 选区/预览 API 全变了

| 旧 | 新 |
|----|----|
| `player_build.previewPose` | 用 `PlayerAction_Build.cursorTarget` / `groundSnappedPos` 等新字段算位姿，或从 `buildPreviews[i].lpos / lrot` 取 |
| `ClearBuildPreviews()` | `BuildTool_Click.ReturnBuildPreviews()` 或清 `buildPreviews` 列表并归还对象池（照搬最新 `BuildTool` 源码的做法） |
| `AddBuildPreview(...)` | 改成 `AddBuildPreviewModel(...)` —— 签名变了，要看 IL 或参考新版其它 mod |
| `cursorWarning` | 新版把提示合进 `BuildTool.cursorText` 一条并用颜色字段区分，没有独立的 warning 字段 |
| `castObjId` | 用 `cursorTarget` / 自己 raycast 取 `objId` |

仍然存在、可直接用的字段：`upgradeLevel`、`cursorText`、`buildPreviews`。

## 推荐参考

别从零改。活跃维护的同类 BuildTool mod 已经把新 API 的用法跑通：

- **`DSP_Mods/CheatEnabler/Patches/FactoryPatch.cs`**（本机位于 `C:\Users\ffqi\dev\c-sharp\DSP_Mods\CheatEnabler\Patches\FactoryPatch.cs`）—— 有完整的 `AddBuildPreviewModel`、`buildPreviews` 管理、`BuildTool` hook 范式，直接照搬 `SelectedBuild.cs` 最快。
- Thunderstore 上 `FastTravelEnabled` / `PickThrough` 等 BuildTool 相关 mod 也可参考。

## 工作量估计

照搬 CheatEnabler 的范式，**一个下午到一天**可重写完。如果从头自己摸 API 变迁，时间翻倍。

## 修完之后

把这个项目加回 CI：编辑 `.github/workflows/ci.yml`，在 `Build AutoStellarNavigate` 后面再加一条

```yaml
- name: Build SelectBuilder
  run: dotnet build SelectBuilder/SelectBuilder.csproj -c Release
```

artifact 的 `path:` 里也加上 `**/bin/Release/**/SelectBuilder.dll`。
