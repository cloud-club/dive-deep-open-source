# Take a look at issue 
- [#42400 aws_pipes_pipe resource does not support target Timestream for Live Analytics](https://github.com/hashicorp/terraform-provider-aws/issues/42400)

## Setting up a development environment
- [Remote Development using SSH](https://code.visualstudio.com/docs/remote/ssh)

<img width="2032" alt="Screenshot 2025-06-08 at 11 12 27 PM" src="https://github.com/user-attachments/assets/feef1607-c6af-4523-af69-9fe8d948d423" />

## TODO
### target_parameters.go

- func targetParametersSchema
  - Add new target parameter schema: Timestream for Live Analytics.
  - Add types, required fields, and validation functions for each member in the type structure.

- func expandPipeTargetParameters
  - Update to include the new target parameter: Timestream for Live Analytics.

- func expandPipeTargetTimestreamParameters
  - Create a new function.

- func flattenPipeTargetParameters
  - Update to include the new target parameter: Timestream for Live Analytics.

- func flattenPipeTargetTimestreamParameters
  - Create a new function.


## Expand and Flatten Functions

- Expand: Transforms data from parameter maps in Terraform to the structure used by the AWS SDK for Go.
- Flatten: Transforms data from the AWS SDK structure to a map format used in the Terraform state.


```
func expandPipeTargetParameters(tfMap map[string]any) *types.PipeTargetParameters {
	if tfMap == nil {
		return nil
	}

	apiObject := &types.PipeTargetParameters{}

	if v, ok := tfMap["batch_job_parameters"].([]any); ok && len(v) > 0 && v[0] != nil {
		apiObject.BatchJobParameters = expandPipeTargetBatchJobParameters(v[0].(map[string]any))
	}

	if v, ok := tfMap["cloudwatch_logs_parameters"].([]any); ok && len(v) > 0 && v[0] != nil {
		apiObject.CloudWatchLogsParameters = expandPipeTargetCloudWatchLogsParameters(v[0].(map[string]any))
	}

...
	return apiObject
}
```

```
func flattenPipeTargetParameters(apiObject *types.PipeTargetParameters) map[string]any {
	if apiObject == nil {
		return nil
	}

	tfMap := map[string]any{}

	if v := apiObject.BatchJobParameters; v != nil {
		tfMap["batch_job_parameters"] = []any{flattenPipeTargetBatchJobParameters(v)}
	}

	if v := apiObject.CloudWatchLogsParameters; v != nil {
		tfMap["cloudwatch_logs_parameters"] = []any{flattenPipeTargetCloudWatchLogsParameters(v)}
	}
...

	return tfMap
}
```
