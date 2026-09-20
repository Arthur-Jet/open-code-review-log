Review complete: 2 finding(s) across 1 selected item(s).

─── .github/workflows/ocr-review-c.yml:87-89 ───
[performance · low] 连通性冒烟测试的超时同步提到 600s（10 分钟）后，当 LLM 端点不可达或挂起时，本应在最前面对外暴露问题的测试步骤最长要等待 10
分钟才失败，显著拖慢失败反馈。且本 workflow 对 dev 分支串行排队（concurrency.cancel-in-progress:
false），一次挂起的连通性测试会阻塞后续排队的评审任务。建议该步骤保留较短超时（如 60~120s）以快速发现连通性问题，600s 仅用于下方真正执行评审、需要长生成时间的步骤。

            OCR_LLM_PROTOCOL: 'anthropic'
-           OCR_LLM_TIMEOUT: '600'
+           OCR_LLM_TIMEOUT: '120'
          run: ocr llm test


─── .github/workflows/ocr-review-c.yml:118-120 ───
[other · low] 单次 LLM 请求超时上限由 3 分钟提高到 10 分钟（约 3.3 倍），而 job 级 timeout-minutes 仍为 45。若 OCR CLI 对较大 diff
会串行发起多次请求（如分块评审），端点持续缓慢或挂起时最坏情况会耗尽整个 45 分钟 job 预算；job 因超时被取消时，连同 if: always()
的汇总/归档步骤也不会执行，评审报告将完全丢失。建议确认 CLI 的分块请求数与重试策略，必要时同步上调 timeout-minutes（如 60），为最坏情形留出余量。


