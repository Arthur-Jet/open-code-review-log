Review complete: 6 finding(s) across 1 selected item(s).

─── .github/workflows/ocr-review-c.yml:204-204 ───
[security · medium] 将 steps.archive.outputs.dest 的 GitHub 表达式直接内联到 run: 脚本属于脚本注入反模式（官方安全加固指南要求表达式只经
env 传入）。当前 DEST 由日期、清洗后的 SAFE（正则会把引号/$/反引号/换行等替换为 -）与十六进制短 SHA 拼成，暂不含 shell
元字符，但这一安全性完全依赖上游清洗逻辑——一旦清洗规则调整或 dest 生成方式变化，注入的命令将以 runner 权限执行。建议改为通过 env: 传递，彻底切断注入面。

-           ARCHIVE_DEST='${{ steps.archive.outputs.dest }}'
+         env:
+           FEISHU_WEBHOOK_URL: ${{ secrets.FEISHU_WEBHOOK_URL }}
+           FEISHU_WEBHOOK_SECRET: ${{ secrets.FEISHU_WEBHOOK_SECRET }}
+           RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
+           ARCHIVE_DEST: ${{ steps.archive.outputs.dest }}
+         run: |
+           # ...
+           if [ -n "$ARCHIVE_DEST" ]; then
+             export ARCHIVE_URL="https://github.com/Arthur-Jet/open-code-review-log/blob/main/${ARCHIVE_DEST}"
+           fi


─── .github/workflows/ocr-review-c.yml:184-186 ───
[bug · medium] 通知步骤失败处理不对称，存在两个缺口：(1) 发送失败时 exit 1 会使整个 job 标红，但评审与归档已成功，通知属非关键环节，宜用
continue-on-error 隔离；(2) 步骤未设 if: always()，一旦上游归档步骤失败，通知会被整体跳过——评审结果既未归档也无任何飞书提醒，通知通道静默丢失。

        - name: Notify Feishu
+         # 通知属非关键环节：归档失败也应尝试通知；通知失败不应拖垮整个 job
+         if: always()
+         continue-on-error: true
          env:
            FEISHU_WEBHOOK_URL: ${{ secrets.FEISHU_WEBHOOK_URL }}


─── .github/workflows/ocr-review-c.yml:195-195 ───
[other · low] 报告缺失时静默 exit 0 且无任何日志，可能掩盖上游评审生成失败，排查时无线索可循。建议补一条 warning 日志说明跳过原因，与上方未配置 webhook 时的
::warning 保持一致的可观测性。

-           [ -f review-result.md ] || exit 0
+           if [ ! -f review-result.md ]; then
+             echo "::warning::review-result.md 不存在（评审失败或无文件变更），跳过飞书通知"
+             exit 0
+           fi


─── .github/workflows/ocr-review-c.yml:200-202 ───
[security · low] 签名密钥经命令行参数 -hmac "$KEY" 传给 openssl，密钥（时间戳+secret 拼接）会出现在进程 argv 中，同一 runner
上的其他进程可通过 ps 或 /proc/<pid>/cmdline 读取。GitHub 托管 runner 为一次性环境，实际风险有限，但更稳妥的做法是在下方已有的 node 脚本内用 crypto
模块计算 HMAC（密钥仅经环境变量读取，不落命令行），并删除 shell 侧的 TS/KEY/SIGN。

-             export TS=$(date +%s)
-             KEY=$(printf '%s\n%s' "$TS" "$FEISHU_WEBHOOK_SECRET")
-             export SIGN=$(openssl dgst -sha256 -hmac "$KEY" -binary < /dev/null | base64)
+             if (process.env.FEISHU_WEBHOOK_SECRET) {
+               const crypto = require("crypto");
+               const ts = Math.floor(Date.now() / 1000).toString();
+               const key = ts + "\n" + process.env.FEISHU_WEBHOOK_SECRET;
+               payload.timestamp = ts;
+               payload.sign = crypto.createHmac("sha256", key).update("").digest("base64");
+             }


─── .github/workflows/ocr-review-c.yml:243-247 ───
[other · low] curl 调用健壮性不足：(1) 未设置 --connect-timeout/--max-time，webhook 网络异常时步骤可能长时间挂起占用 runner；(2)
默认 shell 为 bash -e，curl 连接失败（非零退出）会使步骤立即中断，走不到下方自定义的 ::error 提示与响应诊断；(3) 失败路径上 /tmp/feishu-resp.json
可能不存在或为空，cat/grep 产生误导性报错；(4) 以 grep '"code":0' 精确匹配判定成功偏脆，对返回体格式差异（如带空格或旧版 StatusCode
字段）可能误判。建议补充超时与退出码兜底，并对响应文件读取做容错。

-           HTTP_CODE=$(curl -sS -o /tmp/feishu-resp.json -w '%{http_code}' \
+           HTTP_CODE=$(curl -sS --connect-timeout 10 --max-time 30 \
+             -o /tmp/feishu-resp.json -w '%{http_code}' \
              -X POST -H 'Content-Type: application/json' \
-             -d @/tmp/feishu-payload.json "$FEISHU_WEBHOOK_URL")
-           echo "Feishu response ($HTTP_CODE): $(cat /tmp/feishu-resp.json)"
-           if [ "$HTTP_CODE" != "200" ] || ! grep -q '"code":0' /tmp/feishu-resp.json; then
+             -d @/tmp/feishu-payload.json "$FEISHU_WEBHOOK_URL") || HTTP_CODE=000
+           echo "Feishu response ($HTTP_CODE): $(cat /tmp/feishu-resp.json 2>/dev/null || echo '<无响应体>')"
+           if [ "$HTTP_CODE" != "200" ] || ! grep -qE '"code"[: ]*0|"StatusCode"[: ]*0' /tmp/feishu-resp.json 2>/dev/null; then


─── .github/workflows/ocr-review-c.yml:162-163 ───
[bug · low] dest 输出在 clone/commit/push 之前就写入 GITHUB_OUTPUT，但它派生的「查看归档报告」链接只有在归档真正落到
open-code-review-log/main 后才有效。当前归档失败时通知步骤因缺少 if: always() 而被跳过，问题处于潜伏状态；一旦按建议为通知步骤补充 if:
always()，或归档推送失败后重跑，飞书卡片将生成指向不存在文件的死链。建议将输出移到归档成功路径上：「内容一致跳过提交」的提前退出分支与 push 成功之后各写一次。

-           # 暴露给后续步骤（飞书通知用它生成归档文件链接）
+           if git diff --cached --quiet; then
+             echo "报告内容与已归档版本一致，无需提交"
+             # 归档已存在于 main，链接有效，此处输出 dest
+             echo "dest=$DEST" >> "$GITHUB_OUTPUT"
+             exit 0
+           fi
+           git commit -m "chore: 归档 message-platform 评审报告 ${SAFE}-${SHA7}"
+           # main 期间有新提交时先 rebase 再推；冲突时取本次新内容。
+           # 推送目标是另一个仓库，不会触发本 workflow，无需 [skip ci]
+           git push origin main || { git fetch origin main && git rebase -X theirs origin/main && git push origin main; }
+           # 归档落库成功后再暴露链接，避免通知生成指向不存在文件的死链
            echo "dest=$DEST" >> "$GITHUB_OUTPUT"

