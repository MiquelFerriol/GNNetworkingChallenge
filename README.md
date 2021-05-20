# RouteNet - Graph Neural Networking challenge 2020

#### Organized as part of "ITU AI/ML in 5G challenge"

#### Challenge website: https://bnn.upc.edu/challenge2020

#### Contact mailing list: challenge-kdn@mail.knowledgedefinednetworking.org
(You first need to subscribe at: https://mail.knowledgedefinednetworking.org/cgi-bin/mailman/listinfo/challenge-kdn).

[RouteNet](https://arxiv.org/abs/1901.08113)
is a novel Graph Neural Network (GNN) model that was proposed as a cost-effective alternative to estimate per-source-destination performance metrics (e.g., delay, jitter, loss) in networks. 
Thanks to its GNN architecture that operates over graph-structured data, RouteNet revealed an unprecedented ability to learn and model the complex relationships among topology, routing and input traffic in networks. As a result, it was able to make performance predictions with similar accuracy than resource-hungry packet-level simulators even in network scenarios unseen during training.
This provides network operators with a functional tool able to make accurate predictions of end-to-end Key Performance Indicators (e.g., delay, jitter, loss).

<p align="center"> 
  <img src="/assets/routenet_scheme.png" width="600" alt>
</p>

## Quick Start
### Requirements
We strongly recommend use Python 3.7, since lower versions of Python may cause problems to define all the PATH environment variables.
The following packages are required:
* Tensorflow >= 2.4. You can install it following the official [Tensorflow 2 installation guide.](https://www.tensorflow.org/install)
* NetworkX >= 2.5. You can install it using *pip* following the official [NetworkX installation guide.](https://networkx.github.io/documentation/stable/install.html)
* Pandas >= 0.24. You can install it using *pip* following the official [Pandas installation guide.](https://pandas.pydata.org/pandas-docs/stable/getting_started/install.html)

### Testing the installation
In order to test the installation, we provide a toy dataset that contains some random samples. You can simply verify your installation by executing the [main.py](/code/main.py) file.
```
python main.py
```

You should see something like this:
```
4000/4000 [==============================] - 314s 76ms/step - loss: 67.8137 - MAPE: 67.8137 - val_loss: 150.7025 - val_MAPE: 150.7025
WARNING:tensorflow:Your input ran out of data; interrupting training. Make sure that your dataset or generator can generate at least `steps_per_epoch * epochs` batches (in this case, 5 batches). You may need to use the repeat() function when building your dataset.

Epoch 00001: saving model to ../trained_modelsGNNetworkingChallenge\01-150.70-150.70
```

### Training, evaluation and prediction
We provide two methods, one for training and evaluation and one for prediction. These methods can be found in the [main.py](/code/main.py) file.

#### Training and evaluation
The method for training and evaluation can be found in the [main.py](/code/main.py) file and it is called [train_and_evaluate](/code/main.py#L29). This method trains during a max number of steps and evaluates every number of seconds. 
It also automatically saves the trained model in a directory every time step.

<ins>**IMPORTANT NOTE:**</ins> The execution can be stopped and resumed at any time. However, **if you want to start a new training phase you need to create a new directory (or empty the previous one)**. If you are only doing tests, you can simply pass *model_dir*=None. This will create a temporary directory that will be removed at the end of the execution.

Note that all the parameters needed for the execution (max number of steps, time between model saving...) can be changed in the [[RUN_CONFIG]](code/config.ini#L29) section within the [config.ini](code/config.ini) file.

#### Prediction
In order to make predictions with the trained model, the method [predict](/code/main.py#L66) is provided. This method takes as input the path to the trained model and returns an array with the predictions.
Another method called [predict_and_save](/code/main.py#L90) can also be used in order to make the predictions. In this case, the real and the predicted delays are saved in a CSV file in order to make the visualization of the errors in an easier way. This function returns the Mean Relative Error used for the evaluation. 

## 'How to' guide to modify the code
### Transforming the input data
Now, the model reads the data as it is in the datasets. However, it is often highly recommended to apply some transformations (e.g. normalization, standardization, etc.) to the data in order to facilitate the model to converge during training.
In the [dataset.py](code/read_dataset.py) module you can find a function called [transformation(x, y)](code/read_dataset.py#L125), where the X variable represents the predictors used by the model and the Y variable the target values.
For example, if you want to apply a Min-Max scaling over the 'bandwith' variable, you can use the following code:
```
def transformation(x, y):
    x['bandwith']=(x['bandwith']-bandwith_min)/(bandwith_max-bandwith_min)
    return x, y
```
Where 'bandwith_min' and 'bandwith_max' are respectively the minimum and maximum bandwith values obtained from the dataset.

### Adding new features to the hidden states
Currently, the model only considers the 'bandwith' variable to initialize the initial state of paths.
If we take a look into the [model.py](code/routenet_model.py) module, we can see how the [state initialization](code/routenet_model.py#L99) is done:
```
# Initialize the initial hidden state for links
link_state = tf.concat([
    capacity
], axis=1)

# Initialize the initial hidden state for paths
path_state = tf.concat([
    traffic
], axis=1)
```
For example, if you also want to add the packets transmitted to the paths' initial states, you can easily do so by changing the code to:
```
# Initialize the initial hidden state for links
link_state = tf.concat([
    capacity
], axis=1)

packets = tf.expand_dims(tf.squeeze(inputs['packets']), axis=1)
# Initialize the initial hidden state for paths
path_state = tf.concat([
    traffic,
    packets
], axis=1)
```
Note that now, we changed the shape of the path_state variable that the Embedding takes it as input. So now, we also need to change the input shape of the Embedding:
```
self.path_embedding = tf.keras.Sequential([
            tf.keras.layers.Input(shape=2),
            tf.keras.layers.Dense(int(int(self.config['HYPERPARAMETERS']['path_state_dim']) / 2),
                                  activation=tf.nn.relu),
            tf.keras.layers.Dense(int(self.config['HYPERPARAMETERS']['path_state_dim']), activation=tf.nn.relu)
        ])
```

### Working with Queue Occupancy
One of the main differences that training and test datasets is the link capacities values. This means that, the distribution of the delay is different from the training and the test data. This figure shows the aforementioned different, in it we can find the delay distribution in log scale.

<p align="center"> 
  <img src="/assets/routenet_scheme.PNG" width="600" alt>
</p>

One way of dealing with this problem is to, instead of working with the delay, work with the queue occupancy that is a value that ranges form 0 to 1 and represents the percentage of the queue that is filled with packets. In order to do so, first we need to change the target variable that the model is going to consider. To do so, you need to modify the [read_dataset.py](/code/read_dataset.py#L138) and change this line:
```
}, list(nx.get_node_attributes(D_G, 'delay').values())
```
To this one:
```
 }, list(nx.get_node_attributes(D_G, 'occupancy').values())
```

**IMPORTANT NOTE: ** If you decide to work with the queue occupancy, since now you are working with a link attribute, you will need to change the [readout function](/code/routenet_model.py#L71) to accpet as input the link state, not the path state.

### Available features
In the previous example we could directly include the packets transmitted (i.e., f_['packets']) into the paths’ hidden states. This is because this implementation provides some dataset features that are already processed from the dataset and converted into tensors. Particularly, these tensors are then used to fill a [TensorFlow Dataset structure]( https://www.tensorflow.org/versions/r2.1/api_docs/python/tf/data/Dataset). This can be found in the [read_data.py](/code/read_dataset.py#L175) file, where the following features are included:
* 'bandwidth': This tensor represents the bitrate (bits/time unit) of all the src-dst paths (This is obtained from the traffic_matrix[src,dst][′Flows′][flow_id][‘AvgBw’] values of all src-dst pairs using the [DataNet API]((https://github.com/knowledgedefinednetworking/datanetAPI/tree/challenge2020)))
* 'packets': This tensor represents the rate of packets generated (packets/time unit) of all the src-dst paths (This is obtained from the traffic_matrix[src,dst][′Flows′][flow_id][‘PktsGen’] values of all src-dst pairs using the [DataNet API]((https://github.com/knowledgedefinednetworking/datanetAPI/tree/challenge2020)))
* 'link_capacity': This tensor represents the link capacity (bits/time unit) of all the links found on the network (This is obtained from the topology_object[node][adj][0]['bandwidth'] values of all node-adj pairs using the [DataNet API]((https://github.com/knowledgedefinednetworking/datanetAPI/tree/challenge2020)))
* 'links': This tensor is used to define the message passing in the GNN. Particularly, it uses sequences of link IDs to define the message passing from links to paths.
* 'paths': This tensor is also used in the message passing. Particularly, it uses sequences of path IDs to define the message passing from paths to links.
* 'sequences': This tensor is also used in the message passing. Particularly, it is an auxiliary tensor used to define the order of links in each path.
* 'n_links': This variable represents the number of links in the topology. 
* 'n_paths': This variable represents the total number of src-dst paths in the network.

Note that there are additional features in our datasets that are not included in this TensorFlow Data structure. However, they can be included processing the data with the dataset API and converting it into tensors. For this, you need to modify the [generator()](/code/read_dataset.py#L24) and [input_fn()](/code/read_dataset.py#L138) functions in the [read_dataset.py](/code/read_dataset.py) file. Please, refer to the [API documentation](https://github.com/knowledgedefinednetworking/datanetAPI/tree/challenge2020) of the datasets to see more details about all the data included in our datasets.

**Note:** For the challenge, consider that variables under the *performance_matrix* and the *port_stats* of sample objects (see [API documentation](https://github.com/knowledgedefinednetworking/datanetAPI/tree/challenge2020)) cannot be used as inputs of the model, since they will not be available in the final test set.

### Hyperparameter tunning
If you also want to modify or even add new hyperparameters, you can do so by modifying the [[HYPERPARAMETERS]](code/config.ini#L9) section in the [config.ini](code/config.ini) file.

## Credits
This project would not have been possible without the contribution of:
* [Miquel Ferriol Galmés](https://github.com/MiquelFerriol) - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* [Jose Suárez-Varela](https://github.com/jsuarezv) - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* [Krzysztof Rusek](https://github.com/krzysztofrusek) - Barcelona Neural Networking center, AGH University of Science and Technology
* [Albert López](https://github.com/albert-lopez) - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* [Paul Almasan](https://github.com/paulalmasan) - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* Adrián Manco Sánchez - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* Víctor Sendino Garcia - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* [Pere Barlet Ros](https://github.com/pbarlet) - Barcelona Neural Networking center, Universitat Politècnica de Catalunya
* Albert Cabellos Aparicio - Barcelona Neural Networking center, Universitat Politècnica de Catalunya

## Mailing List
If you have any doubts, or want to discuss anything related to this repository, you can send an email to the mailing list (kdn-users@knowledgedefinednetworking.org). Please, note that you need to subscribe to the mailing list before sending an email ([link](https://mail.knowledgedefinednetworking.org/cgi-bin/mailman/listinfo/kdn-users)).

## License
See [LICENSE](LICENSE) for full of the license text.
```
Copyright Copyright 2020 Universitat Politècnica de Catalunya

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
