あります 👍
CodeBuild のログは基本 CloudWatch Logs に出るので、CLI では CloudWatch Logs 経由で見る形になります。

⸻

✅ CodeBuild のログを見る代表的な方法（CLI）

① ビルドIDからログ情報を取得

まず build id を指定してログの場所を調べます。

aws codebuild batch-get-builds --ids <BUILD_ID> \
  --query 'builds[0].logs.{group:groupName,stream:streamName}' \
  --output table

出力例：
	•	log group: /aws/codebuild/my-project
	•	log stream: xxxxxx

⸻

② CloudWatch Logs でログを読む

最新ログを表示

aws logs get-log-events \
  --log-group-name "<LOG_GROUP>" \
  --log-stream-name "<LOG_STREAM>" \
  --start-from-head


⸻

✅ 「ビルドIDからログを見る」までを一発でやる例

LOG_GROUP=$(aws codebuild batch-get-builds --ids <BUILD_ID> --query 'builds[0].logs.groupName' --output text)
LOG_STREAM=$(aws codebuild batch-get-builds --ids <BUILD_ID> --query 'builds[0].logs.streamName' --output text)

aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LOG_STREAM" \
  --start-from-head


⸻

✅ リアルタイムに近い形で追いかけたい場合

CloudWatch Logs は tail -f みたいに標準では難しいですが、近いことはできます。

aws logs tail "$LOG_GROUP" --follow

※ aws logs tail は AWS CLI v2 で使えます。

⸻

まとめ
	•	CodeBuild は gh run view --log のように1発ではなく
	•	CodeBuildでログの場所を取得
	•	CloudWatch Logsで読む
の2段階です。

⸻

必要なら、codebuild-log <build-id> みたいなコマンド（シェル関数）を作って、gh並みに便利にできますよ。
