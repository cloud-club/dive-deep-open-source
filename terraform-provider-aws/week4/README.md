# Take a issue 

- [#42400 aws_pipes_pipe resource does not support target Timestream for Live Analytics](https://github.com/hashicorp/terraform-provider-aws/issues/42400)

# Example terraform code - target configuration

[1] Target - CloudWatch log group
```
# Amazon EventBridge Pipes
resource "aws_pipes_pipe" "kafka_pipe_4" {
  name     = "SASL_SSL_SHA256_2"
  role_arn = aws_iam_role.pipe_role.arn
  
  source = "smk://10.0.4.20:9093"
  target = aws_cloudwatch_log_group.pipe_logs_3.arn
  
  source_parameters {
    self_managed_kafka_parameters {
      topic_name        = "test"
      
      credentials {
        sasl_scram_256_auth = aws_secretsmanager_secret.kafka_auth.arn
      }

      server_root_ca_certificate = aws_secretsmanager_secret.server_root_ca.arn
      
      vpc {
        security_groups = [aws_security_group.pipe_sg.id]
        subnets         = [data.aws_subnet.existing.id]
      }
    }
  }

  # target configuration 
  target_parameters {
    cloudwatch_logs_parameters {
      log_stream_name = "test"
    }
  }
  
  tags = merge(
    local.default_tags,
    {
      Name = "SASL_SSL_SHA256_2"
    }
  )
}
```

[2] Target - SQS
```
resource "aws_pipes_pipe" "example" {
  name     = "example-pipe"
  role_arn = aws_iam_role.example.arn
  source   = aws_sqs_queue.source.arn
  target   = aws_sqs_queue.target.arn

  source_parameters {
    sqs_queue_parameters {
      batch_size                         = 1
      maximum_batching_window_in_seconds = 2
    }
  }

  target_parameters {
    sqs_queue_parameters {
      message_deduplication_id = "example-dedupe"
      message_group_id         = "example-group"
    }
  }
}
```

# Code
- [target_parameters.go](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/pipes/target_parameters.go)
- [pipe_test.go](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/pipes/pipe_test.go)

# References

- [CreatePipe](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/pipes#Client.CreatePipe)
- [PipeTargetParameters](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/pipes@v1.19.3/types#PipeTargetParameters)
- [PipeTargetTimestreamParameters](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/pipes@v1.19.3/types#PipeTargetTimestreamParameters)
