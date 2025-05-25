# Analyze the Previous PR and Code

## Previous PR
- [feat: Add kms_key_identifier arg for aws_pipes_pipe](https://github.com/hashicorp/terraform-provider-aws/pull/41082)

## Code 

> [internal/service/pipes/pipe.go](https://github.com/acwwat/terraform-provider-aws/blob/main/internal/service/pipes/pipe.go)
  - func `resourcePipe`
    -  Defines the Terraform schema for the aws_pipes_pipe resource.
    -  Associates CRUD operations (create, read, update, delete) with handler functions.
    -  Sets up import support, timeouts, and custom diff logic for tags.
    -  Declares all supported attributes (e.g., name, description, enrichment, source, target, tags) with validation and default values.
  - func `resourcePipeCreate`
    - Handles creation, sets up input, waits for ready state
  - func `resourcePipeRead`
    - Reads state from AWS, updates Terraform state
  - func `resourcePipeUpdate`
    - Handles updates, applies changes, waits for ready state
  - func `resourcePipeDelete`
    - Handles deletion, waits for resource removal


[Resource: aws_pipes_pipe](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/pipes_pipe)

## Files changed in PR 

> [internal/service/pipes/pipe.go](https://github.com/acwwat/terraform-provider-aws/blob/main/internal/service/pipes/pipe.go)
```
func resourcePipe() *schema.Resource {
	return &schema.Resource{
        
        ...

		SchemaFunc: func() map[string]*schema.Schema {
			return map[string]*schema.Schema{
                ...

				"kms_key_identifier": {
					Type:     schema.TypeString,
					Optional: true,
				},       
                ...         
```

```
func resourcePipeCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    ...

	if v, ok := d.GetOk("kms_key_identifier"); ok && v != "" {
		input.KmsKeyIdentifier = aws.String(v.(string))
	}

    ...
```

```
func resourcePipeRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    ...
	d.Set("kms_key_identifier", output.KmsKeyIdentifier)
```

```
func resourcePipeUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    ...
	if d.HasChangesExcept(names.AttrTags, names.AttrTagsAll) {
        ...

		if d.HasChange("kms_key_identifier") {
			input.KmsKeyIdentifier = aws.String(d.Get("kms_key_identifier").(string))
		}
        ...
```



> [internal/service/pipes/pipe_test.go](https://github.com/acwwat/terraform-provider-aws/blob/main/internal/service/pipes/pipe_test.go)

```
func TestAccPipesPipe_kmsKeyIdentifier(t *testing.T) {
	ctx := acctest.Context(t)
	var pipe pipes.DescribePipeOutput
	rName := sdkacctest.RandomWithPrefix(acctest.ResourcePrefix)
	resourceName := "aws_pipes_pipe.test"

	resource.ParallelTest(t, resource.TestCase{
		PreCheck: func() {
			acctest.PreCheck(ctx, t)
			acctest.PreCheckPartitionHasService(t, names.PipesEndpointID)
			testAccPreCheck(ctx, t)
		},
		ErrorCheck:               acctest.ErrorCheck(t, names.PipesServiceID),
		ProtoV5ProviderFactories: acctest.ProtoV5ProviderFactories,
		CheckDestroy:             testAccCheckPipeDestroy(ctx),
		Steps: []resource.TestStep{
			{
				Config: testAccPipeConfig_kmsKeyIdentifier(rName, "${aws_kms_key.test_1.id}"),
				Check: resource.ComposeAggregateTestCheckFunc(
					testAccCheckPipeExists(ctx, resourceName, &pipe),
					acctest.MatchResourceAttrRegionalARN(ctx, resourceName, names.AttrARN, "pipes", regexache.MustCompile(regexp.QuoteMeta(`pipe/`+rName))),
					resource.TestCheckResourceAttrPair(resourceName, "kms_key_identifier", "aws_kms_key.test_1", names.AttrID),
				),
			},
			{
				ResourceName:      resourceName,
				ImportState:       true,
				ImportStateVerify: true,
			},
			{
				Config: testAccPipeConfig_kmsKeyIdentifier(rName, "${aws_kms_key.test_2.arn}"),
				Check: resource.ComposeAggregateTestCheckFunc(
					testAccCheckPipeExists(ctx, resourceName, &pipe),
					acctest.MatchResourceAttrRegionalARN(ctx, resourceName, names.AttrARN, "pipes", regexache.MustCompile(regexp.QuoteMeta(`pipe/`+rName))),
					resource.TestCheckResourceAttrPair(resourceName, "kms_key_identifier", "aws_kms_key.test_2", names.AttrARN),
				),
			},
			{
				Config: testAccPipeConfig_kmsKeyIdentifier(rName, ""),
				Check: resource.ComposeAggregateTestCheckFunc(
					testAccCheckPipeExists(ctx, resourceName, &pipe),
					acctest.MatchResourceAttrRegionalARN(ctx, resourceName, names.AttrARN, "pipes", regexache.MustCompile(regexp.QuoteMeta(`pipe/`+rName))),
					resource.TestCheckResourceAttr(resourceName, "kms_key_identifier", ""),
				),
			},
		},
	})
}
```

```
func testAccPipeConfig_kmsKeyIdentifier(rName, kmsKeyID string) string {
	return acctest.ConfigCompose(
		testAccPipeConfig_base(rName),
		testAccPipeConfig_baseSQSSource(rName),
		testAccPipeConfig_baseSQSTarget(rName),
		fmt.Sprintf(`
resource "aws_kms_key" "test_1" {}
resource "aws_kms_key" "test_2" {}

resource "aws_pipes_pipe" "test" {
  depends_on = [aws_iam_role_policy.source, aws_iam_role_policy.target]

  kms_key_identifier = %[2]q
  name               = %[1]q
  role_arn           = aws_iam_role.test.arn
  source             = aws_sqs_queue.source.arn
  target             = aws_sqs_queue.target.arn
}
`, rName, kmsKeyID))
}
```

## week 4
- [Making Small Changes to Existing Resources](https://hashicorp.github.io/terraform-provider-aws/bugs-and-enhancements/)
- [aws_pipes_pipe resource does not support target Timestream for Live Analytics](https://github.com/hashicorp/terraform-provider-aws/issues/42400)
