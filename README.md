# S3ServerSideEncryptionDemo

!(sample)[sample.png]

## 概要

- S3サーバサード暗号化を検証してみたデモ

## 構成

## デプロイ

```sh
aws cloudformation create-stack \
    --stack-name s3-server-side-encryption-demo \
    --capabilities CAPABILITY_IAM \
    --template-body file://template.yaml
```

### バケット名取得

非暗号化バケット

```sh
NON_ENCRYPTED_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name s3-server-side-encryption-demo \
    --query 'Stacks[].Outputs[?OutputKey==`NonEncryptedBucket`].OutputValue' \
    --output text)
echo ${NON_ENCRYPTED_BUCKET}
  #
```

SSE-S3暗号化バケット

```sh
SSE_S3_ENCRYTED_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name s3-server-side-encryption-demo \
    --query 'Stacks[].Outputs[?OutputKey==`EncryptedBucketBySSES3`].OutputValue' \
    --output text)
echo ${SSE_S3_ENCRYTED_BUCKET}
  #
```

SSE-KMS暗号化バケット

```sh
SSE_KMS_ENCRYTED_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name s3-server-side-encryption-demo \
    --query 'Stacks[].Outputs[?OutputKey==`EncryptedBucketBySSEKMS`].OutputValue' \
    --output text)
echo ${SSE_KMS_ENCRYTED_BUCKET}
  #
```

KMSに作成したキーを利用しSSE-KMS暗号化パケット

```sh
CUSTOMER_SSE_KMS_ENCRYTED_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name s3-server-side-encryption-demo \
    --query 'Stacks[].Outputs[?OutputKey==`EncryptedBucketByCustomerSSEKMS`].OutputValue' \
    --output text)
echo ${CUSTOMER_SSE_KMS_ENCRYTED_BUCKET}
  #
```

### バケットの暗号化設定を確認


```sh
aws s3api get-bucket-encryption \
    --bucket ${NON_ENCRYPTED_BUCKET}
  #
  # An error occurred (ServerSideEncryptionConfigurationNotFoundError) when calling the GetBucketEncryption operation: The server side encryption configuration was not found
```

```sh
aws s3api get-bucket-encryption \
    --bucket ${SSE_S3_ENCRYTED_BUCKET}
  # {
  #     "ServerSideEncryptionConfiguration": {
  #         "Rules": [
  #             {
  #                 "ApplyServerSideEncryptionByDefault": {
  #                     "SSEAlgorithm": "AES256"
  #                 }
  #             }
  #         ]
  #     }
  # }
```

```sh
aws s3api get-bucket-encryption \
    --bucket ${SSE_KMS_ENCRYTED_BUCKET}
  # {
  #     "ServerSideEncryptionConfiguration": {
  #         "Rules": [
  #             {
  #                 "ApplyServerSideEncryptionByDefault": {
  #                     "SSEAlgorithm": "aws:kms"
  #                 }
  #             }
  #         ]
  #     }
  # }
```

```sh
aws s3api get-bucket-encryption \
    --bucket ${CUSTOMER_SSE_KMS_ENCRYTED_BUCKET}
  # {
  #     "ServerSideEncryptionConfiguration": {
  #         "Rules": [
  #             {
  #                 "ApplyServerSideEncryptionByDefault": {
  #                     "KMSMasterKeyID": "a52093d0-b506-4861-87dc-c5cda681c92e",
  #                     "SSEAlgorithm": "aws:kms"
  #                 }
  #             }
  #         ]
  #     }
  # }
```

### ファイルをアップロード

```sh
aws s3api put-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample.png \
    --body sample.png
  # {
  #     "ETag": "\"b420fe104fb7b48b5dcd12de92c444b8\""
  # }
```

```sh
aws s3api put-object \
    --bucket ${SSE_S3_ENCRYTED_BUCKET} \
    --key sample.png \
    --body sample.png
  # {
  #     "ETag": "\"b420fe104fb7b48b5dcd12de92c444b8\"",
  #     "ServerSideEncryption": "AES256"
  # }
```

```sh
aws s3api put-object \
    --bucket ${SSE_KMS_ENCRYTED_BUCKET} \
    --key sample.png \
    --body sample.png
  # {
  #     "SSEKMSKeyId": "arn:aws:kms:ap-northeast-1:1234567890xx:key/8fa790e9-10f6-4d6c-bca1-d9cb5af6e899",
  #     "ETag": "\"8ef45fd3802ab4bb8a8071d2d8d59294\"",
  #     "ServerSideEncryption": "aws:kms"
  # }
```

```sh
aws s3api put-object \
    --bucket ${CUSTOMER_SSE_KMS_ENCRYTED_BUCKET} \
    --key sample.png \
    --body sample.png
  # {
  #     "SSEKMSKeyId": "arn:aws:kms:ap-northeast-1:1234567890xx:key/a52093d0-b506-4861-87dc-c5cda681c92e",
  #     "ETag": "\"1e6befc82805e250db067fd5c648452f\"",
  #     "ServerSideEncryption": "aws:kms"
  # }
```

### SSM-KMSの鍵を確認

SSM-KMSで暗号化した鍵はKMSで管理されているので確認することが可能

デフォルトの場合、SSE-KMSの鍵はAWSで生成されたキーを利用します（KeyManager: **AWS**）

```sh
SSEKMS_KEY_ID=$(aws s3api head-object \
    --bucket ${SSE_KMS_ENCRYTED_BUCKET} \
    --key sample.png \
    --query 'SSEKMSKeyId' \
    --output text)
aws kms describe-key --key-id ${SSEKMS_KEY_ID}
  # {
  #     "KeyMetadata": {
  #         "Origin": "AWS_KMS",
  #         "KeyId": "8fa790e9-10f6-4d6c-bca1-d9cb5af6e899",
  #         "Description": "Default master key that protects my S3 objects when no other key is defined",
  #         "KeyManager": "AWS",
  #         "Enabled": true,
  #         "KeyUsage": "ENCRYPT_DECRYPT",
  #         "KeyState": "Enabled",
  #         "CreationDate": "2018-09-05T06:35:48.668000+00:00",
  #         "Arn": "arn:aws:kms:ap-northeast-1:1234567890xx:key/8fa790e9-10f6-4d6c-bca1-d9cb5af6e899",
  #         "AWSAccountId": "1234567890xx"
  #     }
  # }
```

独自でKMSに作成した鍵（KeyManager: **Customer**）

```sh
CUSTOMER_SSEKMS_KEY_ID=$(aws s3api head-object \
    --bucket ${CUSTOMER_SSE_KMS_ENCRYTED_BUCKET} \
    --key sample.png \
    --query 'SSEKMSKeyId' \
    --output text)
aws kms describe-key --key-id ${CUSTOMER_SSEKMS_KEY_ID}
  # {
  #     "KeyMetadata": {
  #         "Origin": "AWS_KMS",
  #         "KeyId": "a52093d0-b506-4861-87dc-c5cda681c92e",
  #         "Description": "Key created with S3ServerSideEncryptionDemo template.",
  #         "KeyManager": "CUSTOMER",
  #         "Enabled": true,
  #         "KeyUsage": "ENCRYPT_DECRYPT",
  #         "KeyState": "Enabled",
  #         "CreationDate": "2019-11-15T13:51:24.941000+00:00",
  #         "Arn": "arn:aws:kms:ap-northeast-1:1234567890xx:key/a52093d0-b506-4861-87dc-c5cda681c92e",
  #         "AWSAccountId": "1234567890xx"
  #     }
  # }
```

### 非暗号化バケットにオブジェクトをサーバサイド暗号化を指定してアップロード

サーバサイド暗号化にSSE-S3を指定してアップロード

```sh
aws s3api put-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_SSE-S3.png \
    --body sample.png \
    --server-side-encryption AES256
  # {
  #     "ETag": "\"b420fe104fb7b48b5dcd12de92c444b8\"",
  #     "ServerSideEncryption": "AES256"
  # }
```

サーバサイド暗号化にSSE-KMSを指定してアップロード

```sh
aws s3api put-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_SSE-KMS.png \
    --body sample.png \
    --server-side-encryption aws:kms
  # {
  #     "SSEKMSKeyId": "arn:aws:kms:ap-northeast-1:172612068623:key/8fa790e9-10f6-4d6c-bca1-d9cb5af6e899",
  #     "ETag": "\"cebd51529c2fcd566e1724e763047c23\"",
  #     "ServerSideEncryption": "aws:kms"
  # }
```

サーバサイド暗号化にKMSに作成したキーでSSE-KMS暗号化を指定してアップロード

```sh
aws s3api put-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_CustomerSSE-KMS.png \
    --body sample.png \
    --server-side-encryption aws:kms \
    --ssekms-key-id ${CUSTOMER_SSEKMS_KEY_ID}
  # {
  #     "SSEKMSKeyId": "arn:aws:kms:ap-northeast-1:172612068623:key/a52093d0-b506-4861-87dc-c5cda681c92e",
  #     "ETag": "\"e9a3aa7b8af5dd8522edd316c64b46f3\"",
  #     "ServerSideEncryption": "aws:kms"
  # }
```

ユーザが生成したキーを指定してアップロード(SSE-C)

```sh
SSE_CUSTOMER_KEY=$(cat /dev/urandom | base64 -i | fold -w 32 | head -n 1)
echo ${SSE_CUSTOMER_KEY}
  # bKNQl8YfSwa7eKpxSpuamVc+90MQ5BHC

aws s3api put-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_SSE-C.png \
    --body sample.png \
    --sse-customer-algorithm AES256 \
    --sse-customer-key ${SSE_CUSTOMER_KEY}
  # {
  #     "SSECustomerKeyMD5": "rgDjryTvO47GvZmftTvtPw==",
  #     "SSECustomerAlgorithm": "AES256",
  #     "ETag": "\"66aae60ae1a0dd574cf119b899be7463\""
  # }
```

ユーザがキーを指定した場合、複合化する場合には、キーを指定する必要がある

```sh
aws s3api head-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_SSE-C.png
  # 
  # An error occurred (400) when calling the HeadObject operation: Bad Request
```

```sh
aws s3api head-object \
    --bucket ${NON_ENCRYPTED_BUCKET} \
    --key sample_by_SSE-C.png \
    --sse-customer-algorithm AES256 \
    --sse-customer-key ${SSE_CUSTOMER_KEY}
  # {
  #     "AcceptRanges": "bytes",
  #     "ContentType": "binary/octet-stream",
  #     "LastModified": "2019-11-15T14:30:41+00:00",
  #     "ContentLength": 178061,
  #     "SSECustomerAlgorithm": "AES256",
  #     "ETag": "\"66aae60ae1a0dd574cf119b899be7463\"",
  #     "SSECustomerKeyMD5": "rgDjryTvO47GvZmftTvtPw==",
  #     "Metadata": {}
  # }
```
