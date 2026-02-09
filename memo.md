あります。CodeBuild でも AWS CLI で「実行中ビルド一覧」を取れます。
gh run list に相当するのはだいたいこれです。

⸻

✅ CodeBuild版「実行中一覧」

① 実行中のビルドID一覧を取得

aws codebuild list-builds-for-project \
  --project-name <PROJECT_NAME> \
  --sort-order DESC

ただしこれは「全部」出るので、次で絞り込みます。

⸻

② 実行中（IN_PROGRESS）のものだけ見る（実用的）

aws codebuild list-builds-for-project \
  --project-name <PROJECT_NAME> \
  --sort-order DESC \
  --query 'ids[0:20]' \
  --output text

得られたIDを使って詳細取得：

aws codebuild batch-get-builds \
  --ids <BUILD_ID1> <BUILD_ID2> \
  --query 'builds[].{id:id,status:buildStatus,phase:currentPhase,start:startTime}' \
  --output table


⸻

✅ もっと「gh run list」っぽく一発で出す例

IDS=$(aws codebuild list-builds-for-project --project-name <PROJECT_NAME> --sort-order DESC --query 'ids[0:10]' --output text)

aws codebuild batch-get-builds --ids $IDS \
  --query 'builds[].{status:buildStatus,phase:currentPhase,id:id,start:startTime}' \
  --output table


⸻

✅ 実行中だけを抽出したい場合（おすすめ）

IDS=$(aws codebuild list-builds-for-project --project-name <PROJECT_NAME> --sort-order DESC --query 'ids[0:20]' --output text)

aws codebuild batch-get-builds --ids $IDS \
  --query 'builds[?buildStatus==`IN_PROGRESS`].{status:buildStatus,phase:currentPhase,id:id,start:startTime}' \
  --output table


⸻

まとめ
	•	gh run list 相当は
	•	list-builds-for-project
	•	batch-get-builds
の組み合わせで実現できます。
	•	CodeBuild は「一覧コマンド1発」より 2段階取得が基本です。

⸻

必要ならこれを codebuild-run-list みたいなシェル関数にして、ghっぽいUXに整えたものも作れますよ。
