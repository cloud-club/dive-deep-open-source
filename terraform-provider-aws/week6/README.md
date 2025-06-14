# Work log

- func `targetParametersSchema` 
```go
				"timestream_for_live_analytics_parameters": {
					Type:     schema.TypeList,
					Optional: true,
					MaxItems: 1,
					ConflictsWith: []string{
						"target_parameters.0.batch_job_parameters",
						"target_parameters.0.cloudwatch_logs_parameters",
						"target_parameters.0.ecs_task_parameters",
						"target_parameters.0.eventbridge_event_bus_parameters",
						"target_parameters.0.http_parameters",
						"target_parameters.0.kinesis_stream_parameters",
						"target_parameters.0.lambda_function_parameters",
						"target_parameters.0.redshift_data_parameters",
						"target_parameters.0.sagemaker_pipeline_parameters",
						"target_parameters.0.sqs_queue_parameters",
						"target_parameters.0.step_function_state_machine_parameters",
					},
					Elem: &schema.Resource{
						Schema: map[string]*schema.Schema{
							// 1. DimensionMappings
							"dimension_mappings": {
								Type:     schema.TypeList,
								Required: true,
								Elem: &schema.Resource{
									Schema: map[string]*schema.Schema{
										"dimension_name": {
											Type:         schema.TypeString,
											Optional:     true,
											ValidateFunc: validation.StringLenBetween(1, 255),
										},
										"dimension_value": {
											Type:         schema.TypeString,
											Optional:     true,
											ValidateFunc: validation.StringLenBetween(1, 255),
										},
										"dimension_value_type": {
											Type:             schema.TypeString,
											Optional:         true,
											ValidateDiagFunc: enum.Validate[types.DimensionValueType](),
										},
									},
								},
							},
							// 2. TimeValue
							"time_value": {
								Type:         schema.TypeString,
								Required:     true,
								ValidateFunc: validation.StringLenBetween(1, 64),
							},
							// 3. VersionValue
							"version_value": {
								Type:     schema.TypeString,
								Required: true,
								ValidateFunc: validation.All(
									validation.StringLenBetween(1, 64),
									validation.StringMatch(regexache.MustCompile(`^[1-9][0-9]*$`), "must be an integer greater than or equal to 1"),
								),
							},
							// 4. EpochTimeUnit
							"epoch_time_unit": {
								Type:             schema.TypeString,
								Optional:         true,
								ValidateDiagFunc: enum.Validate[types.EpochTimeUnit](),
							},
							// 5. MultiMeasureMappings
							"multi_measure_mappings": {
								Type:     schema.TypeList,
								Required: true,
								Elem: &schema.Resource{
									Schema: map[string]*schema.Schema{
										"multi_measure_attribute_mappings": {
											Type:     schema.TypeString,
											Required: true,
											Elem: &schema.Resource{
												Schema: map[string]*schema.Schema{
													"measure_value": {
														Type:     schema.TypeString,
														Required: true,
													},
													"measure_value_type": {
														Type:             schema.TypeString,
														Required:         true,
														ValidateDiagFunc: enum.Validate[types.MeasureValueType](),
													},
													"multi_measure_attribute_name": {
														Type:         schema.TypeString,
														Required:     true,
														ValidateFunc: validation.StringLenBetween(1, 255),
													},
												},
											},
										},
										"multi_measure_name": {
											Type:         schema.TypeString,
											Required:     true,
											ValidateFunc: validation.StringLenBetween(1, 255),
										},
									},
								},
							},
...
						},
					},
				},
			},
		},
	}
}
```

- func `expandDimensionMapping`: Converts a single Terraform map representation of a dimension mapping into the corresponding AWS API object structure.
```go
func expandDimensionMapping(tfMap map[string]any) *types.DimensionMapping {
	if tfMap == nil {
		return nil
	}

	apiObject := &types.DimensionMapping{}

	if v, ok := tfMap[names.AttrField].(string); ok && v != "" {
		apiObject.DimensionName = aws.String(v)
	}

	if v, ok := tfMap[names.AttrType].(string); ok && v != "" {
		apiObject.DimensionValue = aws.String(v)
	}

	if v, ok := tfMap[names.AttrType].(string); ok && v != "" {
		apiObject.DimensionValueType = types.DimensionValueType(v)
	}

	return apiObject
}
```

- func `expandDimensionMappings`: Transforms a list of Terraform dimension mapping configurations into an array of AWS API dimension mapping objects.
```go
func expandDimensionMappings(tfList []any) []types.DimensionMapping {
	if len(tfList) == 0 {
		return nil
	}

	var apiObjects []types.DimensionMapping

	for _, tfMapRaw := range tfList {
		tfMap, ok := tfMapRaw.(map[string]any)

		if !ok {
			continue
		}

		apiObject := expandDimensionMapping(tfMap)

		if apiObject == nil {
			continue
		}

		apiObjects = append(apiObjects, *apiObject)
	}

	return apiObjects
}
```
