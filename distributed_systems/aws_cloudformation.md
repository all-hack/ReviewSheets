# notes on a cloud guru courses on cloud formation

* what are the basics of cloudformation?
    * templates
        * json file that describes infrastructure
    * stacks
        * related resources managed as a single unit
    * change sets
        * summary of proposed changes to a stack
        * allow you to see how your changes may impact your stacks before implimenting
        
* what are the templates components?
    * format version
        * optional
        * specifies the AWS cloudformation template version that the template conforms to
            * the template format version is not the same as the API or WSDL version
            * the template format version can change independently of the API and WSDL versions
        * latest version is `2010-09-09`
    * description
        * optional
        * must follow the format version
        * text string that describes the template 
    * metadata
        * optional
        * json objects that provide additional information about the template
    * parameters
        * optional
        * specifies values that you can pass in to your template at runtime when creating or updating a stack
        * you can refer to parameters in the resources and outputs section of the template
    * mappings
        * optional
        * a mapping of keys and associated values that you can use to specify conditional parameter values
        * you can match a key to a corresponding value by using the Fn::FindInMap intrinsic function in the resources and ouptuts section
    * conditions
        * optional
        * defines conditions that control whether certain resources are created or whether certain resource properties are assigned a value during stack creation or update
    * outputs
        * optional
        * describes the values that are returned whenever you view your stack's properties
    * resources
        * required
        * specifies the stack resources and their properties, such as an EC2 instance or S3 bucket 
        * you can refer to resources in the resources and outputs  sections of the template        

* what are resource properties?
    * additional options that you can specify for a resource 
        * for each EC2 instance you must specify an AMI ID for that instance
        * declare the AMI ID as a property of the instance
        ```
            "Resources": {
                "MyEC2Instance": {
                    "Type": "AWS::EC2::Instance",
                    "Properties": {
                        "ImageId": "ami-2f726546"
                    }
                }
            }
        ```
    * property values can be
        * literal strings
            * ""
        * list of strings
            * []
        * booleans
        * parameter references
        * pseudo references or the value returned by a function
            * {}

* what are resource attributes?
    * used for advanced scenarios basically defines triggers for infrastructure
    * CreationPolicy
        * associate the CreationPolicy attribute with a resource to prevent its status from reaching create complete until AWS cloudformation receives a specified number of success signals or timeout period is exceeded
    * DeletionPolicy
        * associate the DeletionPolicy attribute with a resource to preserve or backup a resource when its stack is deleted
    * DependsOn
        * associate the DependsOn attribute with a resource to specify that the creation of a specific resource follows another
    * Metadata
        * associate the Metadata attribute with a resources to enable you to associate data structured in the json format with a resource 
    * UpdatePolicy
        * associate the UpdatePolicy attribute with a resources to specify how cloudformation handles updates to the AWS::AutoScaling::AutoScalingGroup resource

* what are intrinsic functions?
    * built in helpers that help you manage stacks
    * Fn::Base64
        * returns the base64 representation of the input string 
        * typically used to pass encoded data to EC2 instances through the UserData property
    * Fn::FindInMap
        * returns the value corresponding to keys in a two-level map that is declared in the Mappings section
    * Fn::GetAtt
        * returns the value of an attribute from a resource in the template
    * Fn::GetAZs
        * returns an array that lists availability zones for a specified region
    * Fn::Join
        * appends a set of values into a single value, separated by the specified delimiter
        * if a delimiter is the empty string, the set of values are concatenated with no delimiter
    * Fn::Select
        * returns a single object from a list of objects by index
    * Ref
        * returns the value of the specified parameter or resource

* what are intrinsic conditional functions?
    * used to conditionally create resources
    * they do exactly what you expect
    * Fn::If
    * Fn::Not
    * Fn::And
    * Fn::Or
    * Fn::Equals
    
* what are pseudo parameters?
    * predefined parameters made by aws, used as arguments for the ref function
    * basically aws constants linked to your account
    * AWS::AccountId
    * AWS::NoValue
    * AWS::Region
    * AWS::NotificationARNs
    * AWS::StackId
    * AWS::StackName

* what are helper scripts?
    * cloudformation provides a set of python helper scripts that you can install software and start services on an EC2 instance
    * the helper scripts run on the EC2 instance as part of the stack creation process    
    * the helper scripts are pre-installed on the latest versions of Amazon Linux AMIs
    * cfn-init
        * used to retrieve metadata
        * installing software
        * configuring files
    * cfn-signal
        * sends a signal to a cloudformation indicating the stack creation policy and a success or timeout
    * cfn-get-metadata
    * cfn-hup
        * used when an update is made to our stack and we need to execute custom hooks

* what are nested stacks?
    * cloud formation stacks referenced from within a stack
        * stacks basically treated as a resource
    * useful for making reusable templates, segmenting resources and avoiding template size limitations
    * launching a template with nested stacks will launch multiple sub-stacks
    * deleting your launching stack will, by default delete all substacks
    * example of nested stacks
    ```
        {
            "AWSTemplateFormatVersion": "2010-09-09",
            "Description": "Template for combining all templates into one nested template for testing.",
            "Parameters": {
                "VpcURL": {
                    "Type": "String",
                    "Default": "https://s3.amazonaws.com/bucket/vpc.template",
                    "Description": "Template URL for BaseNetwork."
                }
            },
            "Resources": {
                "BaseNetwork": {
                    "Type": "AWS::CloudFormation::Stack",
                    "Properties": {
                        "TemplateURL": {"Ref": "VpcURL"},
                        "Parameters": {
                            "CIDRRange": "10.250.0.0"
                        }
                    }
                }
            },
            "Outputs": {
                "BaseNetworkId": {
                    "Value": {"Ref": "BaseNetwork"}
                }
            }
        }
    ```

* what is aws cloudformation designer?
    * a graphic tool for creating, viewing, and modifying cloudformation templates
    * diagram template resources using a drag-and-drop interface
    * edit details using integrated json text editor
















