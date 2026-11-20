# iDICT

面向英语学习者的公开英汉词库，同时提供 JSON 与 CSV，任何前端或脚本都可以直接读取，不需要构建步骤。词表会持续增补，不绑定某个固定词数。

## 文件

```text
manifest.json                    入口清单：版本、词数、文件地址、大小和 SHA-256
words.json                       完整词库（JSON，带分类字段）
words.csv                        完整词库（CSV，同样内容，UTF-8 带 BOM）
LICENSE                          本词库的 MIT 许可全文
licenses/ECDICT-LICENSE.txt      初始词条数据的 MIT 许可全文
```

`words.json` 仍是单个合法 JSON 文档，但已扁平化：顶层只有 `version`、`metadata`、`words` 三个键，`words` 数组里每个词条独占一行，便于 diff、grep 和按行流式处理。

## 如何读取

只需要知道 `manifest.json` 的公开地址：

1. 读取清单，取 `words.path`、`words.bytes`、`words.sha256`、`wordCount`（CSV 对应 `csv.path`、`csv.bytes`、`csv.sha256`）；
2. 相对清单地址解析文件路径并下载；
3. 核对文件大小、SHA-256 和词数，全部吻合再使用。

浏览器中做 SHA-256 校验依赖 `crypto.subtle`，需要 HTTPS 或 localhost。CSV 与 JSON 内容一致，按需要的格式二选一即可。

## 词条字段

```json
{
  "word": "the",
  "phonetic": "ðә",
  "meaning": "art. 这, 那（定冠词，特指已知或唯一的人或事物）",
  "partOfSpeech": "art.",
  "tags": ["常用词", "中考", "高考"],
  "levels": { "collins": 5, "oxford3000": true },
  "definitions": [
    { "partOfSpeech": "art.", "definition": "used before a noun to refer to someone or something already known or already mentioned" },
    { "partOfSpeech": "adv.", "definition": "used before a comparative to show that two things change together (the more, the better)" }
  ],
  "frequency": { "selectionRank": 1, "contemporaryRank": 1, "bncRank": 1 },
  "posClass": ["art."],
  "level": "入门"
}
```

| 字段 | 说明 |
| --- | --- |
| `word` `phonetic` `meaning` | 词头、音标、中文释义（一行一个义项）。 |
| `partOfSpeech` | 词性摘要，可能为空字符串。 |
| `tags` | 词表标签，核心标签 `常用词` 恒在首位，其后是考试标签。 |
| `levels` | 可选，柯林斯星级与牛津 3000 核心词标记。 |
| `forms` | 可选，词形变化（过去式、复数、比较级等）。 |
| `definitions` | 英文释义，已拆成词性与释义两部分。 |
| `sources` | 可选，默认不出现。 |
| `frequency` | `selectionRank` 是在词库中的序号，另两项是当代与 BNC 语料库词频顺序。 |
| `posClass` | 词性分类，规范化后的词性数组，见下文。 |
| `level` | 难度分级，取值 `入门` `基础` `进阶` `高阶`。 |

词条按词频顺序排列，没有词频的排在末尾。

## 分类

两个分类字段都由词条自身的数据推导，不引入外部标注。

### 难度分级 `level`

先按当代语料库词频分档，再对核心学习词放宽一档（牛津 3000 核心词，或柯林斯 4–5 星）：

| 档位 | 词频顺序 | 核心词放宽后 |
| --- | --- | --- |
| `入门` | ≤ 1,500 | — |
| `基础` | ≤ 4,000 | 词频 ≤ 1,500 以外者升一档 |
| `进阶` | ≤ 7,000 | 同上 |
| `高阶` | > 7,000 | 同上 |

当前分布：入门 2,543、基础 1,087、进阶 2,023、高阶 2,347。

### 词性分类 `posClass`

把 `partOfSpeech` 规范化成固定的一组代码，顺序固定为
`n.` `v.` `a.` `adv.` `pron.` `prep.` `conj.` `art.` `num.` `interj.` `aux.` `pref.` `suff.` `abbr.` `其他`。
例如 `vt. / vi. / n.` 会归并为 `["n.", "v."]`——`vt.`、`vi.` 统一记作 `v.`，`pl.`、`sing.` 归入 `n.`，`adj.` 归入 `a.`。

## CSV 格式

`words.csv` 与 `words.json` 内容一致，一行一个词条，UTF-8 带 BOM，可直接用 Excel 打开。数组类字段用 ` | ` 连接，多值字段内部用 `/` 分隔。

| 列 | 对应 JSON | 说明 |
| --- | --- | --- |
| `word` `phonetic` `partOfSpeech` | 同名 | 词头、音标、词性摘要。 |
| `meaning` | `meaning` | 中文释义，多个义项用 ` \| ` 连接。 |
| `level` | `level` | 难度分级。 |
| `posClass` | `posClass` | 词性分类，多个用 ` \| ` 连接。 |
| `tags` | `tags` | 词表标签与考试标签。 |
| `definitions` | `definitions` | 英文释义，格式为 `词性 释义`，多条用 ` \| ` 连接。 |
| `forms` | `forms` | 词形变化，格式为 `键=值`，多项用 `;` 连接，同键多值用 `/` 连接。 |
| `collins` `oxford3000` | `levels` | 柯林斯星级；牛津 3000 为 `true` 或空。 |
| `selectionRank` `contemporaryRank` `bncRank` | `frequency` | 词库内序号、当代词频、BNC 词频。 |

`words.csv` 与 `words.json` 内容完全一致，清单里分别记录了各自的字节数与 SHA-256。仓库不含生成脚本，手工改动数据时需要同时更新这两份文件和清单。

## 数据说明

当前收录 8,000 词，当代语料库词频覆盖 1–10,279；这个数字只是现有规模，不是上限。其中约 7,100 词有柯林斯星级，2,889 词属牛津 3000 核心词，6,350 词记录了词形变化。

这不是任何考试大纲或词典的官方词表，是本站自建的常用词集合。英文多义项与中文义项未逐一对齐，未补写例句，个别义项可能仍有误差。

## 致谢

本词库的初始词条数据来自 **[skywind3000/ECDICT](https://github.com/skywind3000/ECDICT)**——一份由社区长期维护的免费英汉双解词典数据库。音标、中英文释义、词频、考试标签、柯林斯星级、牛津 3000 标记与词形变化，都是从这份数据起步的；没有它的开放与积累，就没有这个项目。

数据取自固定提交 [`39cfb9672c0906b63569e0453c2e6abfb9ab182a`](https://github.com/skywind3000/ECDICT/tree/39cfb9672c0906b63569e0453c2e6abfb9ab182a)，此后由本仓库逐条修订。ECDICT 以 MIT 许可发布，全文见 [licenses/ECDICT-LICENSE.txt](licenses/ECDICT-LICENSE.txt)。

## 许可

本词库以 **MIT 许可**发布，全文见 [LICENSE](LICENSE)，可自由使用、修改、分发，包括商业用途，只需保留版权与许可声明。

初始词条数据另有来源许可：ECDICT 同样以 MIT 发布，其版权与许可声明保留在 [licenses/ECDICT-LICENSE.txt](licenses/ECDICT-LICENSE.txt)，一并随附。
