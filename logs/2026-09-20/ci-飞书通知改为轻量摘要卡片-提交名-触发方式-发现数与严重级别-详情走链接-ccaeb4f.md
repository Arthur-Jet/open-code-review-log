Review complete: 3 finding(s) across 1 selected item(s).

─── .github/workflows/ocr-review-c.yml:199-199 ───
[security · low] 此处将 GitHub 表达式 ${{ steps.range.outputs.mode }} 直接插值进 run 脚本（单引号包裹）。当前 mode
的取值是「Resolve review range」步骤中的三种硬编码常量，暂无被外部输入污染的现实风险，但这种插值模式属于典型的脚本注入隐患：一旦后续 mode
输出改为拼接分支名、提交标题等动态内容，单引号即可被闭合注入任意命令。建议改为在步骤级 env 块中传入（本步骤已有 env 块，与 RUN_URL 等做法保持一致），脚本内直接引用 $MODE
即可；相邻的 SUBJECT 正是通过脚本内生成而非插值，两种风格宜统一为更安全的一种。

- export MODE='${{ steps.range.outputs.mode }}'
+         env:
+           FEISHU_WEBHOOK_URL: ${{ secrets.FEISHU_WEBHOOK_URL }}
+           FEISHU_WEBHOOK_SECRET: ${{ secrets.FEISHU_WEBHOOK_SECRET }}
+           MODE: ${{ steps.range.outputs.mode }}
+         run: |
+           ...
+           # 删除 export MODE='${{ ... }}' 行，node 脚本内 process.env.MODE 直接可用


─── .github/workflows/ocr-review-c.yml:217-224 ───
[maintainability · low] 发现数统计依赖「行首 [类型 · 级别]」这一精确标记格式，且用子串包含关系归类，存在几个失真场景：1) 级别词若出现 critical/严重
等未映射取值，会被计入「其他」桶而非对应严重级别；2) 正则 ^\s*\[([^\]]+)\] 同样会命中行首的 markdown 链接（如 [text](url)），被误计为一条
finding；3) 若报告格式变化导致 0 命中，卡片会展示「发现 0
条」，在存在高危问题时可能误导群内值班同学。建议收紧正则：仅当括号内容确实包含级别词（高/中/低/high/medium/low）时才计数，其余行首方括号行忽略；并可考虑在 total === 0
时展示「未识别到统计标记」之类的兜底文案。

              for (const m of text.matchAll(/^\s*\[([^\]]+)\]/gm)) {
                if (/^ocr\b/i.test(m[1])) continue;
                const sev = m[1].toLowerCase();
-               if (sev.includes("high") || sev.includes("高")) levels.high++;
-               else if (sev.includes("medium") || sev.includes("中")) levels.medium++;
-               else if (sev.includes("low") || sev.includes("低")) levels.low++;
-               else levels.other++;
+               // 仅当括号内容命中级别词才计数，避免把行首 markdown 链接等误计为 finding
+               if (sev.includes("high") || sev.includes("高危") || sev.includes("严重")) levels.high++;
+               else if (sev.includes("medium") || sev.includes("中危")) levels.medium++;
+               else if (sev.includes("low") || sev.includes("低危")) levels.low++;
              }


─── .github/workflows/ocr-review-c.yml:234-235 ───
[security · low] 提交标题来自 git log -1 --pretty=%s，是提交者可控的外部输入。旧实现中它只出现在卡片标题（plain_text，不解析
markdown），本次改为拼入 lark_md 正文后，形如「[点此查看详情](http://evil.example)」的提交标题（60
字符内足够构造）会被飞书渲染为可点击链接，存在卡片内容伪造/诱导点击的风险。建议在拼入 content 前剔除或转义 markdown 元字符（如 []()*_`~），或该行改用 plain_text
元素展示。

+             const subject = (process.env.SUBJECT || "").replace(/[\[\]()*_`~]/g, "");
              const content = [
-               `**提交**：${process.env.SUBJECT}`,
+               `**提交**：${subject}`,

