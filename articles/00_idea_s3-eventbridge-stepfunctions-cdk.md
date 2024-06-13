---
title: "CDK で S3 PUT で Step Functions をトリガーする構成を作る"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [stepfunctions, cdk, s3, eventbridge, aws]
published: false
---

複数の Lambda で Step Functions を構成するときがあります。Step Functions の最初は S3 PUT OBJECT でトリガーしたいときがありませんか？

CDK を使ってそれを実装する一例を説明します。



```ts
import * as cdk from 'aws-cdk-lib';
import * as iam from "aws-cdk-lib/aws-iam";
import * as s3 from "aws-cdk-lib/aws-s3";
import * as sfn from 'aws-cdk-lib/aws-stepfunctions'
import * as s3n from 'aws-cdk-lib/aws-s3-notifications';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as events from 'aws-cdk-lib/aws-events'

export class sfnStack extends cdk.Stack {
  constructor(scope: Construct, id: string) {
    super(scope, id);
    
    const resourceName = "{your_resource_name}"
    
    const videoInput = new s3.Bucket(this, 'videoInput', {
      bucketName: `${resourceName}-video-input`,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
      eventBridgeEnabled: true
    });

    const sfnRole: iam.Role = new iam.Role(this, 'sfnRole', {
      roleName: `${resourceName}-sfnRole`,
      assumedBy: new iam.ServicePrincipal("states.amazonaws.com"),
      path: "/"
    });

    sfnRole.addToPolicy(new iam.PolicyStatement({
      actions: ['*'],
      resources: ['*'],
    }));

    //create statemachine
    const stepFunctions = new sfn.StateMachine(this, 'StateMachineFromFile', {
      definitionBody: sfn.DefinitionBody.fromFile('.{your_definition_body}', {}),
      stateMachineName: `${resourceName}-sfn`,
      role: sfnRole,
    });

    // set event bridge rule to trigger statemachine but put object into video-input bucket
    const eventBridgeRole = new iam.Role(this, 'Role', {
      assumedBy: new iam.ServicePrincipal('events.amazonaws.com'),
    });

    eventBridgeRole.addToPolicy(new iam.PolicyStatement({
      actions: ['*'],
      resources: ['*'],
    }));

    const eventBridgeRule = new events.Rule(this, 's3PutTriggerSfnRule', {
    
      ruleName: `${resourceName}-s3PutTriggerSfnRule`,
      eventPattern: {
        source: ['aws.s3'],
        detailType: ["Object Created"],
        resources: [videoInput.bucketArn],
        detail: {
          "object": {
            "key": [{
              "suffix": ".mp4"
            }]
          }
        }
      }
    })

    eventBridgeRule.addTarget(new targets.SfnStateMachine(stepFunctions, {
      input: events.RuleTargetInput.fromObject({
        bucketName: events.EventField.fromPath('$.detail.bucket.name'),
        objectKey: events.EventField.fromPath('$.detail.object.key'),
      }),
      role: eventBridgeRole
    }))
  }
}
```