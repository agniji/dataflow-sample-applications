# Quickstart

----

*** Note ***

This samples intent is to explore the data engineering needed to work with data generated from a streaming Beam 
pipeline and delivered to an auto encoder - decoder.

It is not intended to demonstrate cutting edge ML model techniques. 

The sample data is very simple repeating pattern, effectively making any train / eval / test void as the data will 
simply repeat across all samples.  

In these samples we use LAST, LAST_TIMESTAMP data generated from the java pipeline.

The training data is without any outliers, the anomaly detection uses data which includes outliers.

----

## Using TF.Example's using a Tensor Flow Extended (TFX) pipeline

* Setup virtual environment
```
virtualenv -p python3.7 streaming-tf-consumer
source streaming-tf-consumer/bin/activate
```

```
git clone https://github.com/GoogleCloudPlatform/dataflow-sample-applications.git
cd dataflow-sample-applications/timeseries-streaming/timeseries-python-applications
pip install -e .
```

From here there are two options: 

* Option 1 : Use a prebuilt [saved_model_example](ml_pipeline_examples/sin_wave_example/saved_model_example) and the prebuilt [transform_graph](ml_pipeline_examples/sin_wave_example/tf_transform_graph_dir)
* Option 2 : Generate bootstrap data to train the model on.

# Option 1

The pre-built model available is in the folder [saved_model_example](ml_pipeline_examples/sin_wave_example/saved_model_example). The model, was built using data generated by [SimpleDataBootstrapGenerator.java](timeseries-java-applications/SyntheticExamples/src/main/java/com/google/dataflow/sample/timeseriesflow/examples/simpledata/transforms/SimpleDataBootstrapGenerator.java).
The data is always repeating, which is a useful property for us to explore the data engineering around the use of the streaming data with TFX. In order to use the model we make use of [RunInferene](https://github.com/tensorflow/tfx-bsl/blob/master/tfx_bsl/beam/run_inference.py) tfx-bsl. 
The Auto encoder / decoder will output values based on the input, these are not predictions. In order to mark a value as anonymous there are two steps that need to be followed:

* The output of the encoder / decoder values have been through normalization, in order to compare the output to the original input the input needs to be independently scaled. This is done by applything the preprocessing_fn directly in Apache Beam to the input. This is done in the transform [ProcessReturn](ml_pipeline/timeseries/encoder_decoder/transforms/process_encdec_inf_rtn.py)
* A threshold needs to be set which defines if the value should be marked as anonymous or not. This value in the sample is arbitrary, for real examples this value will itself need carful analysis and adjustement. Too low a value and there will be too many false positives, too high and the model will miss anomalies.

There are two versions which can be run;

[batch_inference](ml_pipeline_examples/sin_wave_example/inference/batch_inference.py)
* Step 1 - Follow the quick start on the Java README to run [SimpleDataStreamGenerator.java](../timeseries-java-applications/SyntheticExamples/src/main/java/com/google/dataflow/sample/timeseriesflow/examples/simpledata/transforms/SimpleDataStreamGenerator.java) with example_2.
* Step 2 - Run the command with the virtual-env activated, providing values for the location of the ```--saved_model_location``` using the uncompressed provided model, and the location of the generated data from previous step with ```--tfrecord_folder```.
```
python ml_pipeline_examples/sin_wave_example/inference/batch_inference.py --saved_model_location=ml_pipeline_examples/sin_wave_example/saved_model_example/serving_model_dir --tf_transform_graph_dir=ml_pipeline_examples/sin_wave_example/tf_transform_graph_dir --tfrecord_folder=/<your-directory>/simple-data/data/*
```
Dependent on how long you ran the ```SimpleDataStreamGenerator``` you will see outliers in the range of 120 to 300 being detected.

[stream_inference](ml_pipeline_examples/sin_wave_example/inference/stream_inference.py)
* Step 1 - Follow the quick start on the Java README to run [SimpleDataStreamGenerator.java](../timeseries-java-applications/SyntheticExamples/src/main/java/com/google/dataflow/sample/timeseriesflow/examples/simpledata/transforms/SimpleDataStreamGenerator.java) with example_4.
* Step 2 - Create a subscription in the topic, to the read events generated in previous step
* Step 3 - Run the command with the virtual-env activated, providing values for the location of the ```--saved_model_location``` using the uncompressed provided model, and set the subscription to read events from ```--pubsub_subscription```.
```
python ml_pipeline_examples/sin_wave_example/inference/stream_inference.py --saved_model_location=ml_pipeline_examples/sin_wave_example/saved_model_example/serving_model_dir --tf_transform_graph_dir=ml_pipeline_examples/sin_wave_example/tf_transform_graph_dir --pubsub_subscription=projects/<your-project>/subscriptions/<your-subscription>
``` 

While both are running you will see outliers showing around every 50 ticks. 

# Option 2

In order to build the model you will need collect enough data for the model training. 
For this sample a batch generator job [SimpleDataBootstrapGenerator.java](../timeseries-java-applications/SyntheticExamples/src/main/java/com/google/dataflow/sample/timeseriesflow/examples/simpledata/transforms/SimpleDataBootstrapGenerator.java) can also be used. Note this will result in the model learning from a 'perfect' wave with no timestamp jitter, which is not correct but useful for showing the data engineering. 
That job will generate 86400 data points, which will be aggregated to 43200 examples. ( It is recommended you run that job on a production runner like Dataflow rather than the DirectRunner.)

Once that is done, change the information in the [config.py](ml_pipeline_examples/sin_wave_example/config.py) to match your local env.
Run the command with the virtual-env activated:
```
python ml_pipeline_examples/sin_wave_example/timeseries_local_simple_data.py
``` 

This will output a serving_model_dir and a tf_transform_graph under the location you specified for ```PIPELINE_ROOT``` in the config.py file. With this you can now follow the rest of the steps outlines in Option 1 but using your own model.
For example:
```
python MLPipelineExamples/test_pipelines/batch_inference.py --saved_model_location=<PIPELINE_ROOT>/serving_model_dir --tfrecord_folder=/<your-directory>/simple-data/data/*
```

# TODO - Complete instructions for using KFP with the library  
##Using Kubeflow Pipelines
```
tfx pipeline create  --pipeline-path=kfp_timeseries_local_sin_wave.py --endpoint=${ENDPOINT} --build-target-image=${CUSTOM_TFX_IMAGE}
tfx run create --pipeline-name={PIPELINE_NAME} --endpoint=${ENDPOINT}
tfx pipeline update  --pipeline-path=kfp_timeseries_local_sin_wave.py --endpoint=${ENDPOINT}
```
