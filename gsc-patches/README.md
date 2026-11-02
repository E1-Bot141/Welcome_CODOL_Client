# 服务端 GSC 补丁：击杀命中标记显示红 X

本目录存放服务端 GSC 脚本补丁，不属于客户端运行代码，不被 `src/codol_rebuild/` 导入，也不随客户端启动加载。补丁作用于专用服务器（`codol_dedrun.exe`）使用的服务端脚本树。

## 问题

击杀敌人时，命中标记仍显示普通命中的白色材质 `damage_feedback`，没有切换为红色击杀材质 `kill_feedback`。

## 根因

命中标记由 HUD 元素 `hud_damagefeedback` 承载，样式只通过 `setShader` 切换材质。服务端脚本从未产生“击杀”语义：

1. `maps/mp/gametypes/_damage.gsc` 的 `typeHit` 判断链只有 `stun / hitBodyArmor / hitEndGame / standard`，没有判断本次伤害是否致命，因此 `"kill"` 从未被赋值。
2. `maps/mp/gametypes/_damagefeedback.gsc` 没有 `typeHit == "kill"` 分支，也没有预载 `kill_feedback` 材质。

## 改动

| 文件 | 改动 |
| --- | --- |
| `engine/game/scripts/maps/mp/gametypes/_damage.gsc` | `finishPlayerDamage` 内新增致命判断：`finishPlayerDamageWrapper` 已结算本次伤害，若 `!isReallyAlive( victim )`，则 `typeHit = "kill"`。该判断位于判断链最前，保证击杀优先于护甲、残局和眩晕样式。 |
| `engine/game/scripts/maps/mp/gametypes/_damagefeedback.gsc` | `init()` 预载 `kill_feedback`；`updateDamageFeedback()` 新增 `kill` 分支，切换 `kill_feedback` 材质（24×48）并播放 `MP_hit_alert`。命中音效保持不变，本次只改贴图样式。 |

完整修改后的文件位于 `engine/game/scripts/maps/mp/gametypes/`，相对路径与服务端脚本树一致。`0001-kill-hitmarker-red-x.patch` 是同一改动的 unified diff，可直接应用。

标记尺寸使用 24×48，与其它命中标记一致。源图 `kill_feedback.iwi` 为 72×144，由 HUD 按 24×48 绘制。

## 应用

在服务端脚本根目录（即包含 `engine/` 的目录）应用补丁：

```sh
patch -p1 < gsc-patches/0001-kill-hitmarker-red-x.patch
```

或将 `engine/` 下的两个文件按相同相对路径覆盖到服务端脚本树。

## 回滚

```sh
patch -R -p1 < gsc-patches/0001-kill-hitmarker-red-x.patch
```

或覆盖回原始文件。

## 验证

- **无头加载**：以 `CODOL_NCS_TRACE=1` 加载 `mp_killhouse_sh`（dm）。`kill_feedback` 已在服务端继承材质表 `ncs_mat_common_mp` 中，不会新增 `d` 分配，不会触发最终可靠转换失败，也不会因此掉线。
- **机器人对局**：`scr_codol_botfill=4` 下记录 32 / 14 次击杀，零脚本错误；插桩统计分支命中 23 次 `kill`，证明新增分支确实执行。
- **实机对局**：在测试客户端确认红 X 正常显示。

本目录只交付脚本改动。上述验证依赖服务端引擎与客户端实机环境，不能由客户端单元测试替代。

## 文件校验

| 文件 | SHA-256 |
| --- | --- |
| `_damage.gsc` | `3aa99229f20996cb6a9ff668fc564130911aaa684b400e08eac557f92cfbb18b` |
| `_damagefeedback.gsc` | `10ef80f4faa082cafec0cde3798625c2daa27ef4ef3cde2ea5e77cdf11bc078e` |
