Twitter Geolocation Predictor
=============================

This is a deep-learning tool to predict the location of a Twitter user
based solely on the text content of his/her tweets without any other
form of metadata.


Overview
--------

The Twitter Geolocation Predictor is a Recurrent Neural Network
classifier. Every training sample is a collection of tweets labeled with
a location (e.g. country, state, city, etc.). The model will
tokenize all tweets into a sequence of words, and feed them into an
`Embedding Layer <https://en.wikipedia.org/wiki/Word_embedding>`__. The
embeddings will learn the meaning of words and use them as input for two
stacked `Long-Short Term
Memory <http://colah.github.io/posts/2015-08-Understanding-LSTMs/>`__
layers. A `Softmax <https://en.wikipedia.org/wiki/Softmax_function>`__
fully-connected layer at the end yields the classification result.

    
.. image:: https://dl.dropbox.com/s/tvar2ccihtq0ijg/GeoModelGraph.png
   :width: 500px
   :align: center



Getting Started
---------------

Dependencies
~~~~~~~~~~~~
1. Python 3.5
2. tensorflow
3. keras
4. nltk
5. pandas
6. numpy
7. sqlalchemy
8. sklearn
9. psycopg2


Installation
~~~~~~~~~~~~

Clone the repository and install all the dependencies using pip.

.. code:: console

    $ git clone git@github.com:jmatias/uiuc-twitter-geolocation.git
    $ cd uiuc-twitter-geolocation
    $ sudo pip3 install -r requirements.txt

This will install the latest CPU version of Tensorflow. If you would
like to run on a GPU, follow the Tensorflow-GPU `installation
instructions <https://www.tensorflow.org/install/>`__.

Using A Pre-Processed Dataset
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The tool comes with a built-in dataset of ~430K users located in the
U.S. (~410K for training, ~10K for development and ~10K for testing). To
train a model using this dataset, run the train.py sample script.

Note: The dataset has a size of approximately 2.5GB.

.. code-block:: console

    $ python3 train.py --epochs 5 --batch_size 32 --vocab_size 20000 --hidden_size 100 --max_words 100 --classifier state

    Using TensorFlow backend.
    Downloading data from https://dl.dropbox.com/s/ze4ov5j30u9rf5m/twus_test.pickle
    55181312/55180071 [==============================] - 11s 0us/step
    Downloading data from https://dl.dropbox.com/s/kg09i1z32n12o98/twus_dev.pickle
    57229312/57227360 [==============================] - 12s 0us/step
    Downloading data from https://dl.dropbox.com/s/0d4l6jmgguzonou/twus_train.pickle
    2427592704/2427591168 [==============================] - 486s 0us/step

    Building model...
    Hidden layer size: 100
    Analyzing up to 100 words for each sample.
    Building tweet Tokenizer using a 20,000 word vocabulary. This may take a while...
    Tokenizing tweets from 59,546 users. This may take a while...
    Training model...
    Train on 50000 samples, validate on 9546 samples
    Epoch 1/1
        1664/50000 [..............................] - ETA: 3:59 - loss: 3.8578 - acc: 0.0950 - top_5_acc: 0.2536

You can also try using this data from your own source code.

.. code-block:: ipython

    In [1]: from twgeo.data import twus_dataset
    Using TensorFlow backend.

    In [2]: x_train, y_train, x_dev, y_dev, x_test, y_test = twus_dataset.load_state_data()

    In [3]: x_train.shape
    Out[3]: (410336,)

    In [4]: y_train.shape
    Out[4]: (410336,)

    In [5]: x_train, y_train, x_dev, y_dev, x_test, y_test = twus_dataset.load_state_data(size='small')

    In [6]: x_train.shape
    Out[6]: (50000,)

    In [7]: y_train.shape
    Out[7]: (50000,)


Pre-Processing your own data
----------------------------

+------------------------------------------------------------------+------------+
| Tweet Text                                                       | Location   |
+==================================================================+============+
| Hello world! This is a tweet. <eot> This is another tweet. <eot> | Florida    |
+------------------------------------------------------------------+------------+
| Going to see Star Wars tonite!                                   | Puerto Rico|
+------------------------------------------------------------------+------------+
| Pizza was delicious! <eot> I'm another tweeeeeet <eot>           | California |
+------------------------------------------------------------------+------------+


Given a raw dataset stored in a CSV file like the one shown above, we can preprocess said data using :code:`twgeo.data.input.read_csv_data()`. This function will:

    1. Tokenize the tweet text.
    2. Limit repeated characters to a maximum of 2. For example: 'Greeeeeetings' becomes 'Greetings'.
    3. Perform `Porter stemming  <https://en.wikipedia.org/wiki/Stemming>`_ on each token.
    4. Convert each token to lower case.

The location data may be any string or integer value.

.. code:: python

    import twgeo.data.input as input
    tweets, locations = input.read_csv_data('mydata.csv', tweet_txt_column_idx=0, location_column_idx=1)


Training the Model
------------------

.. code:: python

    from twgeo.models.geomodel import Model
    from twgeo.data import twus
    
    # x_train is an array of text. Each element contains all the tweets for a given user. 
    # y_train is an array of integer values, corresponding to each particular location we want to train against.
    x_train, y_train, x_dev, y_dev, x_test, y_test = twus.load_state_data(size='small')

    # num_outputs is the total number of possible classes (locations). In this example, 50 US states plus 3 territories.
    # time_steps is the total number of individual words to consider for each user.
    # Some users have more tweets then others. In this example, we are capping it at a total of 500 words per user.
    geoModel = Model(batch_size=64)
    geoModel.build_model(num_outputs=53, time_steps=500,vocab_size=20000)
                     
    geoModel.train(x_train, y_train, x_dev, y_dev, epochs=5)
    geoModel.save_model('mymodel')

Making Predictions
------------------

.. code:: ipython

    In [1]: from twgeo.models.geomodel import Model
    Using TensorFlow backend.

    In [2]: from twgeo.data import twus_dataset as twus

    In [3]: x_train, y_train, x_dev, y_dev, x_test, y_test = twus.load_state_data(size='small')

    In [4]: geoModel = Model()

    In [5]: geoModel.load_saved_model('mymodel')
    Loading saved model...

    In [6]: geoModel.predict(x_test)
    Out[6]: array(['CA', 'FL', 'NY', ..., 'TX', 'MA', 'KY'], dtype=object)


Results
------------------

The built-in TWUS dataset was used to train US State and US Census Region classifiers. Using a hidden layer size of
300 neurons, timestep window of 500 words and a vocabulary size of 50,000 words, the model achieves the following results.

+-------------------------+-------------------+-----------------------+
| Classification Task     | Test Set Accuracy | Test Set Accuracy @ 5 |
+=========================+===================+=======================+
| US Census Region        | 73.95%            |  N/A                  |
+-------------------------+-------------------+-----------------------+
| US State                | 51.44%            |  75.39%               |
+-------------------------+-------------------+-----------------------+



