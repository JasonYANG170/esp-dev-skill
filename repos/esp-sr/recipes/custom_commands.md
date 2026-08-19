# 自定义中英文命令词

> **适用摘要**: 为 MultiNet5/6/7 自定义中文与英文命令词：通过 `commands_cn.txt`/`commands_en.txt` 文件、`esp_mn_commands_add` API、以及 `tool/multinet_g2p.py` Grapheme-to-Phoneme 工具。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/custom_commands.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "加自己的命令词"
- "中文命令词怎么写"
- "英文命令词 phoneme"
- "multinet_g2p 怎么用"
- "commands_en.txt 格式"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模型 | menuconfig 选了 MultiNet（中/英）并烧录 |
| 工具 | Python + `tool/multinet_g2p.py`（仅 mn5/mn7 英文需要 phoneme 时） |
| 参考文件 | `model/multinet_model/fst/commands_cn.txt`、`commands_en.txt` |

## 命令词格式约束（重要）

- 命令 ID 从 **1** 开始，**不能为 0**
- 同一 ID 可对应多条同义命令
- **不可含阿拉伯数字与特殊字符**
- **中英文不可混用**（同一模型内）
- 单条命令长度限制：`ESP_MN_MIN_PHRASE_LEN`(2) ~ `ESP_MN_MAX_PHRASE_LEN`(63)
- 总命令数上限：`ESP_MN_MAX_PHRASE_NUM`(400)，文档宣称实测稳定支持约 200 条

## 分步说明

### 1. 中文命令词

中文 mn6/mn7 用**拼音**（空格分词）或汉字。仓库默认 `commands_cn.txt` 全部用拼音：

```text
# model/multinet_model/fst/commands_cn.txt 节选（拼音格式）
1,da kai kong tiao
2,guan bi kong tiao
3,da kai dian deng
4,guan bi dian deng
25,da kai kong tiao deng guang
```

直接编辑此文件并重新 `idf.py flash` 即可生效（menuconfig 也可逐条加，但文件更适合批量）。

### 2. 英文命令词（MultiNet6 — grapheme）

mn6 用全大写 grapheme，第三列 phoneme 列可留空（仅为兼容 mn7 格式）：

```text
# commands_en.txt 节选
1,TELL ME A JOKE,TfL Mm c qbK
2,SING A SONG,Sgl c Sel
14,TURN ON THE LIGHT,TkN nN jc LiT
15,TURN OFF THE LIGHT,TkN eF jc LiT
```

### 3. 英文命令词（MultiNet7 — grapheme + 推荐 phoneme）

mn7 推荐填 phoneme 列以提升准确率。grapheme 建议小写（除非缩写发音不同）：

```text
1,tell me a joke,TfL Mm c qbK
2,sing a song,Sgl c Sel
```

若第三列留空，运行时内部 G2P 会兜底，但准确率略降。

### 4. 用 multinet_g2p.py 生成 phoneme

```bash
cd <esp-sr>
python tool/multinet_g2p.py
# 按提示输入英文短语，得到 phoneme 序列，粘贴到 commands_en.txt 第三列
```

> 仅 mn5（必须 phoneme）与 mn7（推荐）需要此步；mn6 用 grapheme 不需要。

### 5. 代码内动态增删（运行时）

```c
#include "esp_mn_speech_commands.h"

esp_mn_commands_clear();

// 中文（mn6/mn7 拼音）
esp_mn_commands_add(1, "da kai dian deng");
esp_mn_commands_add(2, "guan bi chu fang dian deng");
// 多条同 ID
esp_mn_commands_add(3, "kai kong tiao");
esp_mn_commands_add(3, "da kai kong tiao");   // 同 ID=3，两句都触发

// 英文 mn6 grapheme
esp_mn_commands_add(10, "TURN ON THE LIGHT");
// 英文 mn7 显式 phoneme
esp_mn_commands_phoneme_add(11, "SING A SONG", "Sgl c Sel");
// 英文 mn5 必须用 phoneme（grapheme 会被拒）
esp_mn_commands_add(12, "TfL Mm c qbK");   // mn5 整串就是 phoneme

esp_mn_error_t *err = esp_mn_commands_update();
if (err != NULL) {
    for (int i = 0; i < err->num; i++) {
        printf("failed: %s\n", err->phrases[i]->string);
    }
}
```

### 6. 处理 mn5 的差异（旧模型）

mn5 只接受 phoneme（字符表示），命令字符串本身就是 phoneme 序列。区别于 mn6/mn7：

| 模型 | 输入形式 | API |
|---|---|---|
| mn5q8_cn / mn5q8_en | phoneme 字符串 | `esp_mn_commands_add(id, "<phonemes>")` |
| mn6_cn / mn6_en | grapheme（汉字/全大写英文） | `esp_mn_commands_add` |
| mn7_cn / mn7_en | grapheme + 可选 phoneme | `esp_mn_commands_add` / `esp_mn_commands_phoneme_add` |

### 7. 校验命令是否可被 token 化

```c
int ok = multinet->check_speech_command(model_data, "da kai dian deng");
// 非 0 表示可被当前模型解析，0 表示无法解析
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_mn_commands_update` 返回非 NULL | 命令含数字/特殊字符/格式错 | 遍历 `err->phrases` 定位；去掉 `123`、`?`、`.` 等 |
| 中文命令解析失败 | 用了汉字但模型需拼音 / 反之 | mn6/mn7 默认按拼音；确认与 commands_cn.txt 一致 |
| 英文 mn5 报错 | 输入了 grapheme | mn5 只收 phoneme，用 g2p 工具转换 |
| 命令 ID 重复或为 0 | ID 必须 >=1 | ID 从 1 起；同 ID 多句是允许的（同义） |
| 改了 commands_*.txt 不生效 | 没重新 flash | `idf.py flash`（会重新打包 srmodels.bin） |
| 超过 200 条后识别率下降 | 总数过多 | 控制在 200 内；按场景分组、运行时切换 |

## 参考

- `model/multinet_model/fst/commands_cn.txt` — 中文命令词模板（拼音）
- `model/multinet_model/fst/commands_en.txt` — 英文命令词模板（grapheme + phoneme）
- `tool/multinet_g2p.py` — 英文 G2P 工具
- `include/esp32s3/esp_mn_iface.h` — `ESP_MN_MAX_PHRASE_NUM`、`check_speech_command`
- `src/include/esp_mn_speech_commands.h` — `esp_mn_commands_add/phoneme_add/remove/modify/clear/update`
- `docs/en/speech_command_recognition/README.rst` — 各代模型命令词定制方法
- 配套 recipe：`recipes/multinet_commands.md`
