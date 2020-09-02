# これはなに

Data Pipeline で 任意のDynamoDBをS3にバックアップ（1日1回）するためのCloudFormationテンプレート

# 設定方法

## 1.失敗時の通知先とテーブル名を設定
パラメーターセクションで失敗した時の通知先SNSトピックarnとDynamoDBテーブル名を設定  
テーブル名の設定例は3テーブルをバックアップする場合  
4つ以上バックアップしたい場合は数に応じて DynamoDBTableName4: 以降を追加

```
  TopicArn:
    Type: String
    Default: arn:aws:sns:ap-northeast-1:999999999999:XXXXX
  DynamoDBTableName1:
    Type: String
    Default: 【DynamoDBテーブル名1】
  DynamoDBTableName2:
    Type: String
    Default: 【DynamoDBテーブル名2】
  DynamoDBTableName3:
    Type: String
    Default: 【DynamoDBテーブル名3】
```
## 2.開始日付を変更する
変更しないと無駄に実行されるので注意。  
→下記の様に、9/1に設定して9/3に作成すると、Data Pipeline作成後に9/1〜2の2回分が即実行されてしまう
```
              Key: "startDateTime"
              StringValue: "2020-09-01T16:00:00"
```

## 3.Activity設定
パラメーターセクションでDynamoDBTableName1〜3まで作成した場合、  
DynamoDBTableNameXXX（4箇所）をDynamoDBTableName1〜3としたの3つのAcitvityが必要  
```
        - 
          Id: !Ref DynamoDBTableNameXXX
          Name: !Ref DynamoDBTableNameXXX
          Fields: 
            - 
              Key: "type"
              StringValue: "HadoopActivity"
            - 
              Key: "jarUri"
              StringValue: "s3://dynamodb-dpl-ap-northeast-1/emr-ddb-storage-handler/4.11.0/emr-dynamodb-tools-4.11.0-SNAPSHOT-jar-with-dependencies.jar"
            - 
              Key: "runsOn"
              RefValue: "EmrClusterId"
            - 
              Key: "schedule"
              RefValue: "DefaultSchedule"
            - 
              Key: "argument"
              StringValue: "org.apache.hadoop.dynamodb.tools.DynamoDBExport"
            - 
              Key: "argument"
              StringValue: !Join
                             - ""
                             - - "s3://"
                               - !Ref S3BucketBackup
                               - "/"
                               - !Ref DynamoDBTableNameXXX
                               - "/#{format(@scheduledStartTime,'YYYY-MM-dd-HH-mm-ss')}"
            - 
              Key: "argument"
              StringValue: !Ref DynamoDBTableNameXXX
            - 
              Key: "argument"
              StringValue: "0.25"
            - 
              Key: "onFail"
              RefValue: "FailAction"
```

# 実行方法
```
export AWS_PROFILE=【各自の環境に合わせて】
aws cloudformation deploy \
    --stack-name "【スタック名】" \
    --template-file ./datapipeline.yml \
    --s3-bucket "【S3バケット名】"
```
