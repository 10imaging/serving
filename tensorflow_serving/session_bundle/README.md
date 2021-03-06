# TensorFlow Inference Model Format

[TOC]

## Overview

This document describes the data formats and layouts for exporting [TensorFlow]
(https://www.tensorflow.org/) models for inference.

These exports have the following properties,

*   Recoverable
    *   given an export the graph can easily be initialized and run
*   Hermetic
    *   an export directory is self-contained to facilitate distribution

## Directory Structure

~~~
# Directory overview
00000000/
         assets/
         export.meta
         export-?????-of-?????
~~~

*   `00000000` Export version
    *   Format `%08d`
*   `assets` Asset file directory
    *   Holds auxiliary files for the graph (e.g., vocabularies)
*   `export.meta` MetaGraph Definition
    *   Binary [`tensorflow::MetaGraphDef`]
        (https://github.com/tensorflow/tensorflow/tree/master/tensorflow/core/protobuf/meta_graph.proto)
*   `export-?????-of-?????`
    *   Graph Variables
    *   Outputs from Python [`Saver`]
        (https://github.com/tensorflow/tensorflow/tree/master/tensorflow/python/training/saver.py)
        with `sharded=True`.

## Python exporting code

The [`Exporter`](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/session_bundle/exporter.py)
class can be used to export a model in the above format from a Tensorflow python
binary.

## C++ initialization code

The [`LoadSessionBundleFromPath`]
(https://github.com/tensorflow/serving/tree/master/tensorflow_serving/session_bundle/session_bundle.h)
function can be used to create a `tensorflow::Session` and initialize it from an
export. This function takes options and the path to the export and returns a
bundle of export data including a `tensorflow::Session` which can be run.

## Signatures

Graphs used for inference tasks typically have set of inputs and outputs used
at inference time. We call this a signature.

### Standard Signatures (standard usage)

Graphs used for standard inference tasks have standard set of inputs and
outputs. For example, a graph used for a regression task has an input tensor for
the data and an output tensor for the regression values. The signature mechanism
makes it easy to identify the relevant input and output tensors for common graph
applications.

The [`Manifest`](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/session_bundle/manifest.proto)
contains a `Signature` message which contains the task specific inputs and
outputs.

~~~
// A Signature specifies the inputs and outputs of commonly used graphs.
message Signature {
  oneof type {
    RegressionSignature regression_signature = 1;
    ClassificationSignature classification_signature = 2;
    GenericSignature generic_signature = 3;
  }
};
~~~

Standard signature can be set at export time using the `Exporter` API

~~~python
# Run an export.
signature = exporter.classification_signature(input_tensor=input,
                                              classes_tensor=output)
export = exporter.Exporter(saver)
export.init(sess.graph.as_graph_def(),
            default_graph_signature=signature)
export.export(export_path,
              global_step_tensor,
              sess)
~~~

These can be recovered at serving time using utilities in [`signature.h`]
(https://github.com/tensorflow/serving/tree/master/tensorflow_serving/session_bundle/signature.h)

~~~c++
// Get the a classification signature.
ClassificationSignature signature;
TF_CHECK_OK(GetClassificationSignature(bundle->meta_graph_def, &signature));

// Run the graph.
Tensor input_tensor = GetInputTensor();
Tensor classes_tensor;
Tensor scores_tensor;
TF_CHECK_OK(RunClassification(signature, input_tensor, session,
                              &classes_tensor, &scores_tensor));
~~~

### Generic Signatures (custom or advanced usage)

Generic Signatures enable fully custom usage of the `tensorflow::Session` API.
They are recommended for when the standard Signatures do not satisfy a
particular use-case. A general example of when to use these is for a model
taking a single input and generating multiple outputs performing different
inferences.

~~~
// GenericSignature specifies a map from logical name to Tensor name.
// Typical application of GenericSignature is to use a single GenericSignature
// that includes all of the Tensor nodes and target names that may be useful at
// serving, analysis or debugging time. The recommended name for this signature
// is "generic_bindings".
message GenericSignature {
  map<string, TensorBinding> map = 1;
};
~~~

Generic Signatures can be used to compliment a standard signature, for example
to support debugging. Here is an example usage including both the standard
regression signature and a generic signature.

~~~python
named_tensor_bindings = {"logical_input_A": v0,
                         "logical_input_B": v1}
signatures = {
    "regression": exporter.regression_signature(input_tensor=v0,
                                                output_tensor=v1),
    "generic": exporter.generic_signature(named_tensor_bindings)}
export = exporter.Exporter(saver)
export.init(sess.graph.as_graph_def(),
            named_graph_signature=signatures)
export.export(export_path,
              global_step_tensor,
              sess)
~~~

Generic Signature does not differentiate between input and output tensors. It
provides full flexibility to specify the input & output tensors you need.
The benefit is preserving a mapping between names that you specify at export
time (we call these the logical names), and the actual graph node names that may
be less stable and/or auto-generated by TensorFlow.

In [`signature.h`](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/session_bundle/signature.h),
note that the generic signature methods BindGenericInputs and BindGenericNames
are doing simple string to string mapping as a convenience. These methods map
from the names used at training time to actual names in the graph. Use the bound
results from those methods, e.g. `vector<pair<string, Tensor>>` and
`vector<string>` respectively, as inputs to`tensorflow::Session->Run()`. For
`Session->Run()`, map these into the first two parameters, `inputs` and
`output_tensor_names` respectively. The next param, `target_node_names` is
typically null at inference time. The last param outputs is for the results in
the same order of your `output_tensor_names`.

## Initialization

Some graphs many require custom initialization after the variables have been
restored. Such initialization, done through an arbitrary Op, can be added using
the `Exporter` API. If set, `LoadSessionBundleFromPath` will automatically run
the Op when restoring a `Session` following the loading of variables.

## Assets

In many cases we have Ops which depend on external files for initialization
(such as vocabularies). These "assets" are not stored in the graph and are
needed for both training and inference.

In order to create hermetic exports these asset files need to be 1) copied to
each export directory and 2) read when recovering a session from an export base
directory.

Copying assets to the export dir is handled with the callback mechanism. During
an export the export dir is passed to a callback which should copy all assets to
the export directory.

~~~python
      def write_asset(path):
        file_path = os.path.join(path, "file.txt")
        with gfile.FastGFile(file_path, "w") as f:
          f.write("your data here")

      # Gather asset files.
      some_file = tf.constant("file.txt")
      assets = {("file.txt", some_file)}

      # Run an export.
      export = exporter.Exporter(save)
      export.init(sess.graph.as_graph_def(),
                  assets=assets
                  assets_callback=write_asset)
      export.export(export_path,
                    global_step_tensor,
                    sess)
~~~

`AssetFile` binds the name of a tensor in the graph to the name of a file
within the assets directory. `LoadSessionBundleFromPath` will handle the base
path and asset directory swap/concatenation such that the tensor is set with
the fully qualified filename upon return.

# Notes of exporter usage

The typical workflow of model exporting is:

1. Build model graph G
2. Train variables or load trained variables from checkpoint in session S
3. [Optional] build inference graph I
4. Export G

The Exporter should be used as follows:

1. The Saver used in Exporter(saver) should be created under the context of G
2. Exporter.init() should be called under the context of G
3. Exporter.export() should be called using session S
4. If I is provided for Exporter.init(), an exact same Saver should be created
   under I as the saver under G -- in the way that exact same Save/Restore ops
   are created in both G and S
