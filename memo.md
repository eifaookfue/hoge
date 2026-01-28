name: Deploy from ECR

on:
workflow_dispatch:
inputs:
image_tag:
description: "Select image tag (sha-xxxx)"
required: true
type: choice
options: [] # 後で動的に差し替え

jobs:
prepare:
name: Fetch latest ECR tags
runs-on: ubuntu-latest
outputs:
tags: ${{ steps.tags.outputs.tags }}
steps: - name: Configure AWS credentials (OIDC)
uses: aws-actions/configure-aws-credentials@v4
with:
role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
aws-region: ap-northeast-1

      - name: Fetch latest sha-* tags from ECR
        id: tags
        run: |
          TAGS=$(aws ecr describe-images \
            --repository-name myapp \
            --query "sort_by(imageDetails,& imagePushedAt)[-10:].imageTags[]" \
            --output json | jq -r '.[]' | grep '^sha-' || true)

          if [ -z "$TAGS" ]; then
            echo "No sha-* tags found"
            exit 1
          fi

          JSON=$(printf '%s\n' $TAGS | jq -R . | jq -s .)
          echo "tags=$JSON" >> $GITHUB_OUTPUT

deploy:
name: Deploy
needs: prepare
runs-on: ubuntu-latest
steps: - name: Show selected tag
run: |
echo "Selected tag: ${{ github.event.inputs.image_tag }}"

      - name: Resolve digest from ECR
        id: digest
        run: |
          DIGEST=$(aws ecr describe-images \
            --repository-name myapp \
            --image-ids imageTag=${{ github.event.inputs.image_tag }} \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
          echo "Digest: $DIGEST"

      - name: Deploy (example placeholder)
        run: |
          echo "Deploying myapp@${{ steps.digest.outputs.digest }}"
          # ここに ECS/EKS デプロイ処理を入れる

- name: Show recent tags
  run: |
  echo "Recent tags:"
  aws ecr describe-images \
   --repository-name myapp \
   --query "sort_by(imageDetails,& imagePushedAt)[-10:].imageTags[]" \
   --output json | jq -r '.[]'
