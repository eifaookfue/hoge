はい、**できます。しかもそれがベストプラクティスです。**
CIではほぼ全ての現場で

> 最新コミットのSHA → コンテナイメージのタグ

という形を使っています。

---

# 基本形

```
sha-<git commit SHA>
```

例：

```
sha-a1b2c3d4
```

---

# GitHub Actions での実装例（そのまま使えます）

## フルSHAを使う場合

```yaml
- name: Set image tag
  run: |
    echo "IMAGE_TAG=sha-${GITHUB_SHA}" >> $GITHUB_ENV
```

---

## 短縮SHA（おすすめ：8桁）

```yaml
- name: Set image tag
  run: |
    SHORT_SHA=$(echo $GITHUB_SHA | cut -c1-8)
    echo "IMAGE_TAG=sha-${SHORT_SHA}" >> $GITHUB_ENV
```

---

## Docker / Podman ビルド & ECR push

```yaml
- name: Build image
  run: |
    docker build -t myapp:${IMAGE_TAG} .

- name: Login to ECR
  run: |
    aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.ap-northeast-1.amazonaws.com

- name: Push image
  run: |
    docker tag myapp:${IMAGE_TAG} <account>.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:${IMAGE_TAG}
    docker push <account>.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:${IMAGE_TAG}
```

---

# 推奨タグ設計（あなたの文脈向け）

| 種類 | 例             | 用途       |
| ---- | -------------- | ---------- |
| 必須 | `sha-a1b2c3d4` | CD・再現性 |
| 任意 | `v1.2.3`       | リリース   |
| 任意 | `run-12345`    | 再ビルド   |

---

# なぜSHAタグが最適か

| 観点          | 理由                 |
| ------------- | -------------------- |
| 一意性        | 同じコード = 同じSHA |
| 再現性        | ロールバック容易     |
| 追跡          | Gitと1:1対応         |
| 監査          | 変更履歴が明確       |
| ECR immutable | 相性抜群             |

---

# 重要な注意点

## ① 同一コミットを再ビルドすると衝突

対策：

- 原則：再ビルド禁止
  または
- `sha-xxxx-build2` のようにsuffix追加

---

## ② フルSHA vs 短縮SHA

| 種類     | 推奨 |
| -------- | ---- |
| フル40桁 | 過剰 |
| 8〜12桁  | ✅   |

---

# 実務での完成形

CI:

```
myapp:sha-a1b2c3d4
myapp:v1.4.2
```

CD:

```
選択 → sha-a1b2c3d4 → digest解決 → deploy
```

---

# まとめ

- ✅ 最新コミットSHAをタグにできる
- ✅ それが標準
- ✅ ECR immutableと相性最高
- ✅ CD設計が楽になる

---

もしよければ：

- Docker or Podman
- OIDC認証済みか
- マルチアーキ対応要否

に合わせて **完全なCIテンプレ** も作れます。
