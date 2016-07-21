# Terraform Validator

[![Build Status](https://travis-ci.org/elmundio87/terraform_validate.svg?branch=master)](https://travis-ci.org/elmundio87/terraform_validate)

A python package that assists in the enforcement of user-defined standards in Terraform.

The validator uses `pyhcl` to parse Terraform configuration files, then tests the state of the config using custom Assert functions.

## Example Usages

### Check that all AWS EBS volumes are encrypted


```
import terraform_validate

class TestEncryptionAtRest(unittest.TestCase):

    def setUp(self):
        # Tell the module where to find your terraform configuration folder
        self.path = os.path.join(os.path.dirname(os.path.realpath(__file__)),"../terraform")
        self.v = terraform_validate.Validator(self.path)

    def test_aws_ebs_volume(self):
        # Assert that all resources of type 'aws_ebs_volume' are encrypted
        self.v.error_if_property_missing() #Fail any tests if the property does not exist
        self.v.resources('aws_ebs_volume').property('encrypted').equals(True)

    def test_instance_ebs_block_device(self):
        # Assert that all resources of type 'ebs_block_device' that are inside a 'aws_instance' are encrypted
        self.v.error_if_property_missing()
        self.v.resources('aws_instance').property('ebs_block_device').property('encrypted').equals(True)

if __name__ == '__main__':
    suite = unittest.TestLoader().loadTestsFromTestCase(TestEncryptionAtRest)
    unittest.TextTestRunner(verbosity=0).run(suite)

```

```
resource "aws_instance" "foo" {
  # This would fail the test
  ebs_block_device{
    encrypted = false
  }
}

resource "aws_ebs_volume" "bar" {
  # This would fail the test
  encrypted = false
}
```

### Check that AWS resources are tagged correctly

```
import terraform_validate

class TestEncryptionAtRest(unittest.TestCase):

    def setUp(self):
        # Tell the module where to find your terraform configuration folder
        self.path = os.path.join(os.path.dirname(os.path.realpath(__file__)),"../terraform")
        self.v = terraform_validate.Validator(self.path)
        
    def test_aws_ebs_volume(self):
        # Assert that all resources of type 'aws_instance' and 'aws_ebs_volume' have the correct tags
        tagged_resources = ["aws_instance","aws_ebs_volume"]
        required_tags = ["name","version","owner"]
        self.v.resources(tagged_resources).property('tags').has_properties(required_tags)

if __name__ == '__main__':
    suite = unittest.TestLoader().loadTestsFromTestCase(TestEncryptionAtRest)
    unittest.TextTestRunner(verbosity=0).run(suite)
```

## Behaviour functions

These affect the results of the Validation functions in a way that may be required for your tests.

### Validator.error_if_property_missing()

By default, no errors will be raised if a property value is missing on a resource. This changes the behavior of .property() calls to raise an error if a property is not found on a resource.

### Validator.enable_variable_expansion()

By default, variables in property values will not be calculated against their default values. This changes the behaviour of all Validation functions, to work out the value of a string when the variables have default values.

eg. `string = "${var.foo}"` will be read as `string = "1"` by the validator if the default value of `foo` is 1.

## Search functions

These are used to gather property values together so that they can be validated.

### Validator.resources([resource_types])
Searches for all resources of the required types and outputs a `TerraformResourceList`.

Can be chained with a `.property()` function.

Outputs: `TerraformResourceList`

### TerraformResourceList.property(property_name)

Collects all top-level properties in a `TerraformResourceList`  and exposes methods that can be used to validate the property values.

Can be chained with another `.property()` call to fetch nested properties.

eg. ``.resource('aws_instance').property('name')``

### TerraformResourceList.find_property(regex)

Similar to `TerraformResourceList.property()`, except that it will attempt to use a regex string to search for the property.

eg. ``.resource('aws_instance').find_property('tag[a-z]')``


### TerraformPropertyList.property(property_name)

Collects all nested properties in `TerraformPropertyList` and exposes methods that can be used to validate the property values.

eg. ``.resource('aws_instance').property('tags').property('name')``


### TerraformPropertyList.find_property(regex)

Similar to `TerraformPropertyList.property()`, except that it will attempt to use a regex string to search for the property.

eg. ``.resource('aws_instance').find_property('tag[a-z]')``

## Validation functions

If there are any errors, these functions will print the error and raise an AssertionError. The purpose of these functions is to validate the property values of different resources.

### TerraformResourceList.has_properties([required_properties])

Will raise an AssertionError if any of the properties in `required_properties` are missing from a `TerraformResourceList`.

### TerraformPropertyList.has_properties([required_properties])

Will raise an AssertionError if any of the properties in `required_properties` are missing from a `TerraformPropertyList`.

### TerraformResourceList.name_matches_regex(regex)

Will raise an AssertionError if the Terraform resource name does not match the value of `regex`

### TerraformPropertyList.equals(expected_value)

Will raise an AssertionError if the value of the property does not equal `expected_value`

### TerraformPropertyList.not_equals(unexpected_value)

Will raise an AssertionError if the value of the property equals `unexpected_value`

### TerraformPropertyList.matches_regex(regex)

Will raise an AssertionError if the value of the property does not match the value of `regex`
  
