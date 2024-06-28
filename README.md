How to enable monitoring in your prompt flow using Azure **Python SDK**?

**Reference:** [Product Documentation link](https://learn.microsoft.com/en-us/azure/ai-studio/how-to/monitor-quality-safety?tabs=python#configure-monitoring)

**1.** Create a temporary directory and within it create a file named `enable-monitoring.py` with the following content:

[**enable-monitoring.py**](enable-monitoring.py)

```
from azure.ai.ml import MLClient
from azure.ai.ml.entities import (
    MonitorSchedule,
    CronTrigger,
    MonitorDefinition,
    ServerlessSparkCompute,
    MonitoringTarget,
    AlertNotification,
    GenerationTokenStatisticsMonitorMetricThreshold,
    GenerationTokenStatisticsSignal,
    GenerationSafetyQualityMonitoringMetricThreshold,
    GenerationSafetyQualitySignal,
    BaselineDataRange,
    LlmData,
)
from azure.ai.ml.entities._inputs_outputs import Input
from azure.ai.ml.constants import MonitorTargetTasks, MonitorDatasetContext

# Authentication package
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Update your azure resources details
subscription_id = "INSERT YOUR SUBSCRIPTION ID"
resource_group = "INSERT YOUR RESOURCE GROUP NAME"
workspace_name = "INSERT YOUR WORKSPACE NAME" # This is the same as your AI Studio project name
endpoint_name = "INSERT YOUR ENDPOINT NAME" # This is your deployment name without the suffix (e.g., deployment is "contoso-chatbot-1", endpoint is "contoso-chatbot")
deployment_name = "INSERT YOUR DEPLOYMENT NAME"
aoai_deployment_name ="INSERT YOUR AOAI DEPLOYMENT NAME"
aoai_connection_name = "INSERT YOUR AOAI CONNECTION NAME"

# These variables can be renamed but it is not necessary
app_trace_name = "app_traces"
app_trace_Version = "1"
monitor_name ="gen_ai_monitor_both_signals" 
defaulttokenstatisticssignalname ="token-usage-signal" 
defaultgsqsignalname ="gsq-signal"

# Determine the frequency to run the monitor, and the emails to recieve email alerts
trigger_schedule = CronTrigger(expression="15 10 * * *")
notification_emails_list = ["test@example.com", "def@example.com"]

ml_client = MLClient(
    credential=credential,
    subscription_id=subscription_id,
    resource_group_name=resource_group,
    workspace_name=workspace_name,
)

spark_compute = ServerlessSparkCompute(instance_type="standard_e4s_v3", runtime_version="3.3")
monitoring_target = MonitoringTarget(
    ml_task=MonitorTargetTasks.QUESTION_ANSWERING,
    endpoint_deployment_id=f"azureml:{endpoint_name}:{deployment_name}",
)

# Set thresholds for passing rate (0.7 = 70%)
aggregated_groundedness_pass_rate = 0.7
aggregated_relevance_pass_rate = 0.7
aggregated_coherence_pass_rate = 0.7
aggregated_fluency_pass_rate = 0.7

# Create an instance of gsq signal
generation_quality_thresholds = GenerationSafetyQualityMonitoringMetricThreshold(
    groundedness = {"aggregated_groundedness_pass_rate": aggregated_groundedness_pass_rate},
    relevance={"aggregated_relevance_pass_rate": aggregated_relevance_pass_rate},
    coherence={"aggregated_coherence_pass_rate": aggregated_coherence_pass_rate},
    fluency={"aggregated_fluency_pass_rate": aggregated_fluency_pass_rate},
)
input_data = Input(
    type="uri_folder",
    path=f"{endpoint_name}-{deployment_name}-{app_trace_name}:{app_trace_Version}",
)
data_window = BaselineDataRange(lookback_window_size="P7D", lookback_window_offset="P0D")
production_data = LlmData(
    data_column_names={"prompt_column": "question", "completion_column": "answer", "context_column": "context"},
    input_data=input_data,
    data_window=data_window,
)

gsq_signal = GenerationSafetyQualitySignal(
    connection_id=f"/subscriptions/{subscription_id}/resourceGroups/{resource_group}/providers/Microsoft.MachineLearningServices/workspaces/{workspace_name}/connections/{aoai_connection_name}",
    metric_thresholds=generation_quality_thresholds,
    production_data=[production_data],
    sampling_rate=1.0,
    properties={
        "aoai_deployment_name": aoai_deployment_name,
        "enable_action_analyzer": "false",
        "azureml.modelmonitor.gsq_thresholds": '[{"metricName":"average_fluency","threshold":{"value":4}},{"metricName":"average_coherence","threshold":{"value":4}}]',
    },
)

# Create an instance of token statistic signal
token_statistic_signal = GenerationTokenStatisticsSignal()

monitoring_signals = {
    defaultgsqsignalname: gsq_signal,
    defaulttokenstatisticssignalname: token_statistic_signal,
}

monitor_settings = MonitorDefinition(
compute=spark_compute,
monitoring_target=monitoring_target,
monitoring_signals = monitoring_signals,
alert_notification=AlertNotification(emails=notification_emails_list),
)

model_monitor = MonitorSchedule(
    name = monitor_name,
    trigger=trigger_schedule,
    create_monitor=monitor_settings
)

ml_client.schedules.begin_create_or_update(model_monitor)

print('\nDone creating monitor schedule.')
```

**2.** Update the Python code with your deployment data

**2.1.** Update resource names in line 24.

```
# Update your azure resources details
subscription_id = "INSERT YOUR SUBSCRIPTION ID"
resource_group = "INSERT YOUR RESOURCE GROUP NAME"
workspace_name = "INSERT YOUR WORKSPACE NAME" # This is the same as your AI Studio project name
endpoint_name = "INSERT YOUR ENDPOINT NAME" # This is your deployment name without the suffix (e.g., deployment is "contoso-chatbot-1", endpoint is "contoso-chatbot")
deployment_name = "INSERT YOUR DEPLOYMENT NAME"
aoai_deployment_name ="INSERT YOUR AOAI DEPLOYMENT NAME"
aoai_connection_name = "INSERT YOUR AOAI CONNECTION NAME"
```

Example:
```
# Update your azure resources details
subscription_id = "9788a92c-2f71-4629-8173-7ad449cb50e1"
resource_group = "rg-paulolacerdawsp"
workspace_name = "paulolacerda-4000"
endpoint_name = "paulolacerda-4000-njdnb"
deployment_name = "paulolacerda-4000-njdnb-1"
aoai_deployment_name ="gpt-4"
aoai_connection_name = "ai-paulowspai975059219091_aoai"
```

**2.2.** Update flow field names

In line 75 of the file, you will find the mapping of fields used in the flow to the fields used in monitoring (`prompt_column`, `completion_column`, `context_column`).

Replace them as necessary according to the fields of your flow.

```
production_data = LlmData(
    data_column_names={"prompt_column": "question", "completion_column": "answer", "context_column": "context"},
    input_data=input_data,
    data_window=data_window,
)
```

Example based on Exercise 4 from the LLMOps Workshop:
```
production_data = LlmData(
    data_column_names={"prompt_column": "question", "completion_column": "answer", "context_column": "documents"},
    input_data=input_data,
    data_window=data_window,
)
```

*Save* the file and go to the next step.

**3.** Open a terminal within the newly created folder.

**3.1.** Install required libraries

```
pip install azure-ai-ml
pip install azure-identity
```

**3.2.** Login to azure (if not logged in yet)

```
az login
```

For more options on az login run `az login --help`

**3.3.** Run the python program

```
python enable-monitoring.py
```

**Done creating monitor schedule!**