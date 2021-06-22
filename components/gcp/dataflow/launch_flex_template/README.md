
# Name
Data preparation by using a flex template to submit a job to Cloud Dataflow.

# Labels
GCP, Cloud Dataflow, Kubeflow, Pipeline

# Summary
A Kubeflow Pipeline component to prepare data by using a flex template to submit a job to Cloud Dataflow.

# Details

## Intended use
Use this component when you have a pre-built Cloud Dataflow flex template and want to launch it as a step in a Kubeflow Pipeline.

## Runtime arguments
Argument        | Description                 | Optional   | Data type  | Accepted values | Default    |
:---            | :----------                 | :----------| :----------| :----------     | :----------|
project_id | The ID of the Google Cloud Platform (GCP) project to which the job belongs. | No | GCPProjectID |  |  |
container_spec_gcs_path | Gcs path to a file with json serialized ContainerSpec as content. | No  | GCSPath  |  |  |
launch_parameters | The parameters that are required to launch the template. The schema is defined in [LaunchFlexTemplateParameter](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/projects.locations.flexTemplates/launch#LaunchFlexTemplateParameter). The parameter `jobName` is replaced by a generated name. | Yes  |  Dict | A JSON object which has the same structure as [LaunchFlexTemplateParameter](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/projects.locations.flexTemplates/launch#LaunchFlexTemplateParameter) | None |
location | The regional endpoint to which the job request is directed.| Yes  |  GCPRegion |    |  None |
staging_dir |  The path to the Cloud Storage directory where the staging files are stored. A random subdirectory will be created under the staging directory to keep the job information. This is done so that you can resume the job in case of failure.|  Yes |  GCSPath |   |  None |
validate_only | If True, the request is validated but not executed.   |  Yes  |  Boolean |  |  False |
wait_interval | The number of seconds to wait between calls to get the status of the job. |  Yes  | Integer  |   |  30 |

## Input data schema

The input `container_spec_gcs_path` must contain a valid Cloud Dataflow flex template. The template can be created by following the instructions in [Using Flex Templates](https://cloud.google.com/dataflow/docs/guides/templates/using-flex-templates).

## Output
Name | Description
:--- | :----------
job_id | The id of the Cloud Dataflow job that is created.

## Caution & requirements

To use the component, the following requirements must be met:
- Cloud Dataflow API is enabled.
- The component can authenticate to GCP. Refer to [Authenticating Pipelines to GCP](https://www.kubeflow.org/docs/gke/authentication-pipelines/) for details.
- The Kubeflow user service account is a member of:
    - `roles/dataflow.developer` role of the project.
    - `roles/storage.objectViewer` role of the Cloud Storage Object `container_spec_gcs_path.`
    - `roles/storage.objectCreator` role of the Cloud Storage Object `staging_dir.`



```python
%%capture --no-stderr

!pip3 install kfp --upgrade
```

2. Load the component using KFP SDK


```python
import kfp.components as comp

dataflow_flex_template_op = comp.load_component_from_url(
    'https://raw.githubusercontent.com/kubeflow/pipelines/master/components/gcp/dataflow/launch_flex_template/component.yaml')
help(dataflow_flex_template_op)
```

### Sample

Note: The following sample code works in an IPython notebook or directly in Python code.
In this sample, we run a Google-provided word count template from `gs://dataflow-templates/latest/flex/Streaming_Data_Generator`. The template takes a json schema file as input and outputs streaming data to a Cloud Storage bucket.

#### Set sample parameters


```python
# Required Parameters
PROJECT_ID = '<Please put your project ID here>'
GCS_SCHEMA_LOCATION = 'gs://<Please put your GCS path to the JSON schema file here>' # No ending slash
```


```python
# Optional Parameters
EXPERIMENT_NAME = 'Dataflow - Launch Flex Template'
OUTPUT_PATH = '{}/out/sdg'.format(GCS_WORKING_DIR)
```

#### Example pipeline that uses the component


```python
import kfp.dsl as dsl
import json
@dsl.pipeline(
    name='Dataflow launch flex template pipeline',
    description='Dataflow launch flex template pipeline'
)
def pipeline(
    project_id = PROJECT_ID,
    container_spec_gcs_path = 'gs://dataflow-templates/latest/flex/Streaming_Data_Generator',
    launch_parameters = json.dumps({
       'parameters': {
           'schemaLocation': GCS_SCHEMA_LOCATION,
           'sinkType': 'GCS',
           'outputDirectory': OUTPUT_PATH,
           'qps': 1,
       }
    }),
    location = '',
    validate_only = 'False',
    staging_dir = GCS_WORKING_DIR,
    wait_interval = 30):
    dataflow_flex_template_op(
        project_id = project_id,
        container_spec_gcs_path = container_spec_gcs_path,
        launch_parameters = launch_parameters,
        location = location,
        validate_only = validate_only,
        staging_dir = staging_dir,
        wait_interval = wait_interval)
```

#### Compile the pipeline


```python
pipeline_func = pipeline
pipeline_filename = pipeline_func.__name__ + '.zip'
import kfp.compiler as compiler
compiler.Compiler().compile(pipeline_func, pipeline_filename)
```

#### Submit the pipeline for execution


```python
#Specify pipeline argument values
arguments = {}

#Get or create an experiment and submit a pipeline run
import kfp
client = kfp.Client()
experiment = client.create_experiment(EXPERIMENT_NAME)

#Submit a pipeline run
run_name = pipeline_func.__name__ + ' run'
run_result = client.run_pipeline(experiment.id, run_name, pipeline_filename, arguments)
```

#### Inspect the output


```python
!gsutil cat $OUTPUT_PATH*
```

## References

* [Component python code](https://github.com/kubeflow/pipelines/blob/master/components/gcp/container/component_sdk/python/kfp_component/google/dataflow/_launch_flex_template.py)
* [Component docker file](https://github.com/kubeflow/pipelines/blob/master/components/gcp/container/Dockerfile)
* [Sample notebook](https://github.com/kubeflow/pipelines/blob/master/components/gcp/dataflow/launch_flex_template/sample.ipynb)
* [Cloud Dataflow Templates overview](https://cloud.google.com/dataflow/docs/guides/templates/overview)

## License
By deploying or using this software you agree to comply with the [AI Hub Terms of Service](https://aihub.cloud.google.com/u/0/aihub-tos) and the [Google APIs Terms of Service](https://developers.google.com/terms/). To the extent of a direct conflict of terms, the AI Hub Terms of Service will control.
