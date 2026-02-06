はい。いちばん簡単なサンプルは「ALB 経由で Green 側にだけ届くテスト用URL を叩いて、200なら SUCCEEDED、それ以外は FAILED を返す」Lambda です。
ECS のライフサイクルフックは Lambda が hookStatus を返さない／失敗するとロールバック扱いになります。 ￼

以下、最短で動く形をまとめます。

⸻

1) まず“テスト用の到達経路”を用意する（超重要）

Blue/Green では「本番トラフィック」と「テストトラフィック」を分けます。
最も簡単なのは ALB のリスナールールで分岐することです。

例（どれか1つでOK）：
	•	パスで分ける：/deploy-check だけ Green 側TGへ
	•	ヘッダで分ける：X-Test-Traffic: true があれば Green 側TGへ

これができていないと、Lambda が叩いているのが Blue なのか Green なのか分からなくなります。

⸻

2) Lambda（最小）コード例：HTTP 200なら成功、他は失敗

Python（標準ライブラリだけ）で書くとこうです。

import os
import urllib.request
import urllib.error

URL = os.environ["TEST_URL"]  # 例: http://xxxxx.elb.amazonaws.com/deploy-check
TIMEOUT = float(os.environ.get("TIMEOUT_SECONDS", "3"))

def lambda_handler(event, context):
    try:
        req = urllib.request.Request(
            URL,
            headers={"X-Test-Traffic": "true"}  # ヘッダ分岐するなら
        )
        with urllib.request.urlopen(req, timeout=TIMEOUT) as res:
            status = res.getcode()

        # 200系だけ合格（必要なら条件を増やす）
        if 200 <= status < 300:
            return {"hookStatus": "SUCCEEDED"}
        else:
            return {"hookStatus": "FAILED"}

    except Exception:
        # 例外は失敗扱い（＝ロールバック）
        return {"hookStatus": "FAILED"}

ECS ライフサイクルフックの戻り値は hookStatus: SUCCEEDED | FAILED が基本です。 ￼

⸻

3) Lambda の作り方（最短手順）
	1.	Lambda 作成（ランタイム：Python）
	2.	環境変数を設定
	•	TEST_URL：テスト用に Green に届くURL（ALBのURL＋パス等）
	•	（任意）TIMEOUT_SECONDS：3 など
	3.	タイムアウト：まず 5〜10秒くらい（短め推奨）

⸻

4) ネットワーク注意（ALBが “internal” の場合だけ）
	•	ALB が public なら、Lambda を VPC 外のままでも叩けることが多いです
	•	ALB が internal なら、Lambda を 同じVPCのサブネットに入れて、
	•	Lambda の SG → ALB の SG に到達できる（インバウンド許可）
	•	Lambda に NAT 経由の外向きが必要かはあなたの処理次第（このサンプルは不要）

⸻

5) IAM注意（ハマりどころ）

ECS が Lambda を呼べないと、フック失敗でロールバックになります。 ￼
（権限不足の典型エラーがドキュメントにも載っています）
	•	ECS 側の実行ロール（フック呼び出し役）に lambda:InvokeFunction が必要
	•	Lambda 実行ロールには AWSLambdaBasicExecutionRole（CloudWatch Logs）でOK

⸻

6) どのフックに付けるのが一番ラク？

最初の検証なら、だいたいこのどちらかが分かりやすいです：
	•	TEST_TRAFFIC_SHIFT（テスト用トラフィックがGreenへ）
	•	POST_SCALE_UP（Greenタスク起動後）

まずは「Green に届くテスト用URL」が叩けるかを確認しやすいフックに付けるのが成功率高いです。
（※フックはロールバック時に再実行されることがあるので、処理は冪等にするのが安全です） ￼

⸻

次に、あなたの環境に合わせて“最短の設定”を提案できます

差し支えなければ、これだけ教えてください（質問じゃなく、貼ってくれればOK）：
	•	ALB は internal ですか？ internet-facing ですか？
	•	いま リスナールールで Green にだけ届く経路（パス or ヘッダ）は作れそうですか？

この2点が分かれば、TEST_URL の具体形（パス/ヘッダ）と、SGの通し方まで“最短構成”に落として書きます。